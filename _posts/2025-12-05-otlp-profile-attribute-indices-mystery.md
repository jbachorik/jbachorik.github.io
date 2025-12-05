---
layout: post
title: "Debugging the OTLP Profile Attribute Indices Mystery"
date: 2025-12-05
categories: profiling otlp protobuf debugging
tags: profiling otlp protobuf debugging
---

This is the story of how a handful of integers managed to gaslight three separate tools, a Docker container, one human, and a very unimpressed cat on a radiator, while rain hammered the windows and the logs scrolled by like sleet.

## The Problem

On a dark, wet evening, while implementing sample attributes support for the JFR-to-OTLP profiles converter, we ran into a maddening validation issue.

- Our generated OTLP profiles:
  - Passed `protoc` validation (the official Protocol Buffers compiler)
- But:
  - `profcheck` insisted our `attribute_indices` were out of range

The errors looked like this:

```text
sample[0]: attribute_indices: [0]: index 2 is out of range [0..2)
sample[1]: attribute_indices: [0]: index 3 is out of range [0..2)
sample[2]: attribute_indices: [0]: index 4 is out of range [0..2)
...
sample[99]: attribute_indices: [0]: index 101 is out of range [0..2)
```

We expected `1` for every sample (only one real attribute in the table). Instead, we got a bleak little staircase: 2, 3, 4, …, 101.

That pattern screams “you’re reading the wrong field.” But of course, before blaming the tools, we had to prove we were not just sleep-deprived and wrong.

The cat remained neutral.

## Initial Investigation

### Step 1: Sanity-Check Internal Data

First step: confirm our own in-memory state was sane.

```java
System.out.println("AttributeTable size: " + attributeTable.size());
System.out.println("Sample attributeIndices: " + Arrays.toString(sample.attributeIndices));
```

Output:

```text
AttributeTable size: 2  // [0]=sentinel, [1]=sample.type:cpu
Sample attributeIndices: [1]
```

So internally:

- Attribute table: 2 entries (sentinel at 0, one real attribute at 1)
- Each sample: `attributeIndices = [1]`

Nothing obviously broken on our side. Just the usual glow of a terminal in a dark room.

### Step 2: Inspect the Wire

Next: see what we actually wrote to disk.

```bash
hexdump -C /tmp/debug_cpu.pb | head -50
```

We saw patterns like:

```text
08 01 10 01 18 02 22 01 01 2a 08 ...
08 01 10 01 18 03 22 01 01 2a 08 ...
08 01 10 01 18 04 22 01 01 2a 08 ...
```

Decoding:

- `08` = (1 << 3) | 0 → field 1, wire type 0 (varint) → `stack_index`
- `01` = value 1

- `10` = (2 << 3) | 0 → field 2, wire type 0 (varint)
- `01` = value 1

- `18` = (3 << 3) | 0 → field 3, wire type 0 (varint)
- `02/03/04` = incrementing values (looked like `link_index`)

- `22` = (4 << 3) | 2 → field 4, wire type 2 (length-delimited)
- `01 01` = packed length 1, value [1]

Field 2 appeared as a plain varint, not a packed repeated field (wire type 2). That didn’t match what our code claimed to do:

```java
encoder.writePackedVarintField(
    OtlpProtoFields.Sample.ATTRIBUTE_INDICES, sample.attributeIndices);
```

So either:

1. Our encoder lied, or  
2. The bytes we were staring at weren’t from the code we thought they were

The cat yawned, voting for “you’re looking at the wrong file.”

### Step 3: Old Debug File Trap

The hex dump was from an old debug file, generated before some refactoring.

After cleaning up, rebuilding, and generating a fresh `.pb`, the wire format matched what we expected; the hex dump reflected the correct packed encoding, and `protoc` happily decoded it.

So at this point:

- Fresh profile → OK  
- Our own tests → OK  
- `protoc` → OK  
- `profcheck` → still complaining

We’d left the realm of “broken encoding” and entered “why does this tool live in another universe?”

Outside, it was still raining. Inside, the cat had gone back to sleep.

### Step 4: Ask Protoc, the Referee

We wired `protoc` in explicitly:

```bash
protoc --decode=opentelemetry.proto.profiles.v1development.ProfilesData     --proto_path=/proto/opentelemetry-proto     opentelemetry/proto/profiles/v1development/profiles.proto     < profile.pb
```

Result:

- ✅ `protoc` decoded cleanly
- ✅ The structure matched the proto definition
- ✅ `attribute_indices` were `[1]` as intended

So:

- According to the official protobuf implementation and the proto we pointed it to, the payload was fine.

Yet `profcheck` was adamant that `attribute_indices` contained 2, 3, 4, …, 101 — way outside `[0..2)`.

Which means `profcheck` wasn’t just “unhappy”; it was looking at a different field map.

### Step 5: Suspecting Profcheck’s View of the World

`profcheck` reporting incrementing values as `attribute_indices` is very specific:

- That pattern matched what we expected in `link_index`, not `attribute_indices`.

Hypothesis:

> `profcheck` is decoding our `link_index` field as `attribute_indices` because it’s using a different field numbering.

In other words, we were speaking “trunk proto” and `profcheck` was speaking “some older proto,” and both sides just pretended it was fine.

## The Breakthrough

The key clue came from inspecting the Go module docs:

> https://pkg.go.dev/go.opentelemetry.io/proto/otlp/profiles/v1development#Sample

The generated Go type looked like:

```go
type Sample struct {
    StackIndex         int32    `protobuf:"varint,1,opt,name=stack_index"`
    Values             []int64  `protobuf:"varint,2,rep,packed,name=values"`
    AttributeIndices   []int32  `protobuf:"varint,3,rep,packed,name=attribute_indices"`
    LinkIndex          int32    `protobuf:"varint,4,opt,name=link_index"`
    TimestampsUnixNano []uint64 `protobuf:"fixed64,5,rep,packed,name=timestamps_unix_nano"`
}
```

Field numbers according to the Go module:

1. `stack_index` = 1  
2. `values` = 2  
3. `attribute_indices` = 3  
4. `link_index` = 4  
5. `timestamps_unix_nano` = 5  

Then we looked at the proto on GitHub (trunk):

```protobuf
message Sample {
  int32 stack_index = 1;
  repeated int32 attribute_indices = 2;
  int32 link_index = 3;
  repeated int64 values = 4;
  repeated fixed64 timestamps_unix_nano = 5;
}
```

Field numbers in trunk:

1. `stack_index` = 1  
2. `attribute_indices` = 2  
3. `link_index` = 3  
4. `values` = 4  
5. `timestamps_unix_nano` = 5  

So:

- Trunk proto: `attribute_indices=2`, `link_index=3`, `values=4`
- Go module: `values=2`, `attribute_indices=3`, `link_index=4`

That’s not just cosmetic. That’s “ABI incompatible if you mix them” territory.

We had implemented against **trunk**.

`profcheck` was built against the **Go module**, which is pegged to an older commit with a different field layout.

Result:

- We encode:
  - `field 2` → `attribute_indices`
  - `field 3` → `link_index`
  - `field 4` → `values`
- Profcheck decodes with:
  - `field 2` → `values`
  - `field 3` → `attribute_indices`
  - `field 4` → `link_index`

So:

- Our `link_index` staircase (2, 3, 4, …, 101) lands in `profcheck`’s `attribute_indices`
- It checks that against an attribute table of size 2
- And complains: “index 101 is out of range [0..2).”

We weren’t mis-encoding. We were just talking to a tool frozen at a different commit, like trying to use current TLS with a browser from 2004.

The cat, if asked, would have called this “classic human mistake: not checking which food bag you opened.”

## The Root Cause

The real issue was:

> The proto definition in the **Go module** used by `profcheck` did not match the **trunk** version in the GitHub repo. The module was pinned to an older commit with different field numbering.

Summarizing:

- Our encoding:
  - Spec-compliant against trunk `.proto`
  - Accepted by `protoc` using that trunk schema
- Profcheck:
  - Built against a Go module generated from an older proto commit
  - Interpreted our fields using its own (older) field numbers
  - Misread `link_index` as `attribute_indices` and complained accordingly

So the bug wasn’t in protobuf, or in our encoder, or in the cat.

It was straightforward schema drift hidden behind the same message name.

## The Fix

From our side, the pragmatic move was to align with the field layout that the Go module (and therefore `profcheck`) uses.

```java
// Sample fields
public static final class Sample {
  public static final int STACK_INDEX = 1;
  public static final int VALUES = 2;               // Was 4
  public static final int ATTRIBUTE_INDICES = 3;    // Was 2
  public static final int LINK_INDEX = 4;           // Was 3
  public static final int TIMESTAMPS_UNIX_NANO = 5; // Unchanged

  private Sample() {}
}
```

After this change, our encoding matched what `profcheck` actually expects, given the Go module it’s compiled against.

### Results

After rebuilding:

- ✅ `protoc` validation: PASSED  
- ✅ `profcheck` validation: PASSED  

The remaining `profcheck` warnings are about timestamp ranges in test data, which is a separate problem and not related to field numbering.

The attribute index saga was resolved. The rain kept going. The cat demanded food.

## Lessons Learned

### 1. “Spec-Compliant” Needs a Version

You don’t just need “the spec”; you need **the exact spec revision** your tools are using:

- GitHub trunk proto != “what your ecosystem is actually using”
- Go modules can be pinned to older commits with incompatible field numbering

If different components silently use different schema versions, you’ll get “everyone is correct locally” and “everything is broken together.”

### 2. Protoc Is Necessary but Not Sufficient

`protoc` will tell you whether your payload is valid for the `.proto` you supply.

It does **not** tell you whether:

- That `.proto` matches what your validators, agents, or services were generated from
- Your tools have silently drifted to different commits

You still need to check version alignment across the whole chain.

### 3. Hex Dumps Still Earn Their Keep

Looking at the wire format is still worth it:

- Confirms actual field numbers and wire types
- Helps spot patterns like “incrementing values where you expected constants”
- Proves whether the bug is in your encoder or in someone else’s schema

It’s not glamorous, but it cuts through a lot of guesswork. Think of it as staring into a binary snowstorm and finding the footprints.

### 4. Old Artifacts and Old Schemas Are Both Traps

Two equally annoying sources of confusion:

- Stale `.pb` files lying around from a previous build
- Stale schema versions baked into modules and tools

You have to check both. Otherwise you’re just chasing ghosts in a cold CI log.

### 5. Incrementing Values in a “Constant” Field = Red Flag

If you expect:

- `attribute_indices = [1]` everywhere

and you see:

- 2, 3, 4, … 101

assume:

- You’re not looking at the field you think you are; you’re decoding someone else’s field under the wrong schema.

Treat that as a schema mismatch until proven otherwise.

## Debugging Checklist for Protobuf Chaos

When protobuf starts acting like January weather, go through this:

1. ✅ Log internal data structures (sizes, indices, contents)  
2. ✅ Validate with `protoc` using the **exact proto** you think you implement  
3. ✅ Hex dump the payload and decode tags/wire types for suspicious fields  
4. ✅ Compare:
   - GitHub trunk `.proto`
   - The `.proto` or generated files actually used by your build
   - The generated code used by your tooling (Go/Java/etc.)  
5. ✅ Identify what commit / version your external tools (e.g. `profcheck`) are built from  
6. ✅ Look for patterns in the “wrong” values (increments, offsets, constants)  
7. ✅ Regenerate artifacts to avoid inspecting stale files  
8. ✅ Assume **version skew** before assuming “protobuf is broken”

## Implementation Details

### Dual Validation

We kept both validators in play:

```bash
# 1. Protoc validation (against chosen proto version)
protoc --decode=opentelemetry.proto.profiles.v1development.ProfilesData     --proto_path=.     opentelemetry/proto/profiles/v1development/profiles.proto     < profile.pb

# 2. Profcheck validation (what the Go tooling ecosystem sees)
profcheck profile.pb
```

Interpretation:

- If `protoc` fails → your payload doesn’t match the schema you’re claiming.  
- If `protoc` passes and `profcheck` fails → either:
  - You have semantic issues (timestamps, ranges, etc.), or
  - The schema that `profcheck` was generated from does not match your `.proto`.

### Dockerized Validation Environment

To keep everything reproducible:

```dockerfile
FROM golang:1.23-alpine AS builder
# Build profcheck from OpenTelemetry sig-profiling repo
RUN git clone https://github.com/open-telemetry/sig-profiling.git
WORKDIR /build/sig-profiling/tools/profcheck
RUN go build -o /profcheck .

FROM alpine:latest
RUN apk add --no-cache protobuf protobuf-dev git

WORKDIR /proto
RUN git clone --depth=1 https://github.com/open-telemetry/opentelemetry-proto.git

COPY --from=builder /profcheck /usr/local/bin/profcheck

RUN cat > /usr/local/bin/validate-profile << 'EOF'
#!/bin/sh
# Run both validators...
EOF
```

One image, fixed versions, fewer surprises when CI runs at 3 a.m. and the logs look like frozen rain.

## Conclusion

The root cause was not exotic:

> The proto definition in the **Go module** used by `profcheck` was pinned to an older OTLP profiles commit, with a different field numbering than the **trunk** proto we used.

Everything else followed automatically:

- Our payload was correct for trunk.
- `protoc` confirmed it.
- `profcheck`, using the older schema, misinterpreted our `link_index` as `attribute_indices` and complained about “out of range” indices.

The fix was brutally simple: update four field constants to match the module’s schema. The hard part was realizing we were arguing with a tool that had been compiled against yesterday’s reality.

The practical takeaway:

- Don’t just check “the proto.”
- Check which **commit** and which **generated code** your tools are actually using.
- When two validators disagree on a cold, wet night, suspect schema version skew first — then everything else.

The cat, for the record, was right to ignore all of it.
