---
layout: post
title: "Debugging the OTLP Profile Attribute Indices Mystery"
date: 2025-12-05
categories: [profiling, otlp, protobuf, debugging]
tags: [opentelemetry, protobuf, otlp, profiling, debugging, go, java]
---

This is the story of how a handful of integers managed to gaslight three separate tools, a Docker container, one human, and a very unimpressed cat on a radiator, while rain hammered the windows and the logs scrolled by like sleet.

The short version:  
our OTLP profiles were **perfectly valid** according to `protoc`, but `profcheck` insisted our `attribute_indices` were out of range.  
The long version is below. It involves:

- An evening that felt like November at 16:30
- Two different proto schemas with the same message name
- Field numbers quietly rearranged between commits
- A cat that refused to care

---

## 1. The Symptom: Profcheck vs. Reality

We were adding **sample attributes support** to an OTLP profiles converter. The flow was:

1. Convert internal data → OTLP `ProfilesData`
2. Serialize to protobuf
3. Validate using:
   - `protoc` (canonical protobuf implementation)
   - `profcheck` (OpenTelemetry profile validator)

The results:

- `protoc`: ✅ all good  
- `profcheck`: ❌ screaming about `attribute_indices`

The errors looked like:

```text
sample[0]: attribute_indices: [0]: index 2 is out of range [0..2)
sample[1]: attribute_indices: [0]: index 3 is out of range [0..2)
sample[2]: attribute_indices: [0]: index 4 is out of range [0..2)
...
sample[99]: attribute_indices: [0]: index 101 is out of range [0..2)
```

We expected every sample to reference a **single attribute** at index `1`. Instead, we got a nice ascending staircase: `2, 3, 4, …, 101`.

On a good day, that pattern would be annoying. On a cold, wet evening with terminal light reflecting off the window and the cat side-eyeing the radiator, it was downright offensive.

---

## 2. Context: What We Thought We Were Encoding

We had a simple model:

- `AttributeTable`
  - Index `0`: sentinel
  - Index `1`: `"sample.type: cpu"`
- Each sample:
  - `attribute_indices = [1]`

Quick debug logging:

```java
System.out.println("AttributeTable size: " + attributeTable.size());
System.out.println("Sample attributeIndices: " + Arrays.toString(sample.attributeIndices));
```

Output:

```text
AttributeTable size: 2  // [0]=sentinel, [1]=sample.type:cpu
Sample attributeIndices: [1]
```

So in memory:

- Table size is correct
- Index is correct
- Everything looks boringly sane

This is the point where you start suspecting the **wire format**.

---

## 3. Wire Format Autopsy

### 3.1. First Hex Dump (The Red Herring)

We dumped the file:

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

- `08` = `(1 << 3) | 0` → field **1**, wire type **0** (varint) → `stack_index`
- `01` = value **1**

- `10` = `(2 << 3) | 0` → field **2**, wire type **0** (varint)
- `01` = value **1**

- `18` = `(3 << 3) | 0` → field **3**, wire type **0** (varint)
- `02/03/04` = incrementing values (looked like `link_index`)

- `22` = `(4 << 3) | 2` → field **4**, wire type **2** (length-delimited)
- `01 01` = packed length **1**, value `[1]`

At first glance, this suggested:

- Field 2 was being emitted as a **single varint**, not a **packed repeated field**
- That clashed with our encoder call:

  ```java
  encoder.writePackedVarintField(
      OtlpProtoFields.Sample.ATTRIBUTE_INDICES, sample.attributeIndices);
  ```

So either:

1. The encoder was misbehaving, or  
2. We were looking at the wrong file

The cat, being more experienced with humans than protobuf, silently voted for (2).

### 3.2. The Stale Artifact

That hex dump was from an **old debug file** generated before recent refactoring.

After:

```bash
rm /tmp/debug_cpu.pb
# rebuild + rerun generator
hexdump -C /tmp/debug_cpu.pb | head -50
```

…the wire format now matched the expected packed encoding, and `protoc` decoding aligned perfectly with our structures.

So:

- Fresh `.pb` → ✔  
- Our own decoder → ✔  
- `protoc` → ✔  
- `profcheck` → still ❌

Verdict: the problem was not “we wrote garbage.” It was “someone else is reading it differently.”

Outside, the rain kept going. Inside, the cat fell asleep. We moved on.

---

## 4. Calling in Protoc as Referee

We wired in a canonical decode using the **trunk** OTLP profiles proto:

```bash
protoc --decode=opentelemetry.proto.profiles.v1development.ProfilesData \
    --proto_path=/proto/opentelemetry-proto \
    opentelemetry/proto/profiles/v1development/profiles.proto \
    < profile.pb
```

Result:

- `protoc` decoded without complaint
- The decoded `ProfilesData` matched the trunk proto layout
- `attribute_indices` were `[1]` everywhere

So for the schema we pointed `protoc` at:

- Our payload was **100% spec-compliant**
- Our data matched expectations

At this point, there were only two realistic options:

1. `profcheck` is buggy  
2. `profcheck` is using a **different schema** than the one we’re validating against

Option (2) is more boring and much more likely.

---

## 5. Profcheck’s Reality: The Go Module

The real turning point came from looking at the Go module docs:

> <https://pkg.go.dev/go.opentelemetry.io/proto/otlp/profiles/v1development#Sample>

The generated Go struct:

```go
type Sample struct {
    StackIndex         int32    `protobuf:"varint,1,opt,name=stack_index"`
    Values             []int64  `protobuf:"varint,2,rep,packed,name=values"`
    AttributeIndices   []int32  `protobuf:"varint,3,rep,packed,name=attribute_indices"`
    LinkIndex          int32    `protobuf:"varint,4,opt,name=link_index"`
    TimestampsUnixNano []uint64 `protobuf:"fixed64,5,rep,packed,name=timestamps_unix_nano"`
}
```

Field numbers **according to the Go module**:

1. `stack_index` = **1**  
2. `values` = **2**  
3. `attribute_indices` = **3**  
4. `link_index` = **4**  
5. `timestamps_unix_nano` = **5**  

Now compare that to the proto from GitHub **trunk**:

```protobuf
message Sample {
  int32 stack_index = 1;
  repeated int32 attribute_indices = 2;
  int32 link_index = 3;
  repeated int64 values = 4;
  repeated fixed64 timestamps_unix_nano = 5;
}
```

Field numbers **in trunk**:

1. `stack_index` = **1**  
2. `attribute_indices` = **2**  
3. `link_index` = **3**  
4. `values` = **4**  
5. `timestamps_unix_nano` = **5**  

Let’s put that into a table.

### 5.1. Field Number Mismatch

| Logical field         | Trunk proto (GitHub) | Go module (profcheck) |
|-----------------------|----------------------|-----------------------|
| `stack_index`         | 1                    | 1                     |
| `attribute_indices`   | 2                    | 3                     |
| `link_index`          | 3                    | 4                     |
| `values`              | 4                    | 2                     |
| `timestamps_unix_nano`| 5                    | 5                     |

So:

- In trunk, `attribute_indices` = **2**
- In the Go module, `values` = **2**, `attribute_indices` = **3**

Someone reshuffled the field numbers between commits.  
The Go module was pegged to an **older layout**, while trunk had a newer one.

We had implemented against **trunk**.  
`profcheck` was compiled against the **Go module**.

Result:

- We wrote:

  - Field **2** → `attribute_indices`
  - Field **3** → `link_index`
  - Field **4** → `values`

- `profcheck` decoded:

  - Field **2** → `values`
  - Field **3** → `attribute_indices`
  - Field **4** → `link_index`

So from `profcheck`’s point of view:

- Our `link_index` staircase (2, 3, 4, …, 101) appeared in **its** `attribute_indices` field
- It checked those against an attribute table of size **2**
- And fairly yelled: “index 101 is out of range [0..2).`

Both sides were internally consistent.  
They just didn’t agree on the meaning of **field number 2+**.

---

## 6. The Real Root Cause

The core issue was:

> The proto definition in the **Go module** used by `profcheck` did not match the **trunk** version in the GitHub repo.  
> The Go module was pinned to an **older commit** where the field numbering was different.

Thus:

- Our encoding:
  - Correct for **trunk** schema
  - Verified by `protoc` using that schema
- Profcheck’s decoding:
  - Based on an **older schema**
  - Interpreted our tags with its own field map
  - Misread `link_index` as `attribute_indices`

No exotic protobuf edge case.  
No subtle encoder bug.  
Just plain schema drift masquerading as “validation errors.”

---

## 7. The Fix: Align with Profcheck’s Schema

From a practical standpoint, we had two options:

1. Fight `profcheck` and enforce trunk schema everywhere  
2. Align our field numbering with the schema that the ecosystem is actually using right now (the Go module)

We chose option 2. The code change was almost embarrassingly simple:

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

After that:

- Our encoder wrote field numbers matching the **Go module layout**
- `profcheck` and `protoc` both interpreted the payload consistently

### 7.1. Validation After the Change

We re-ran our checks:

```bash
# Canonical validation
protoc --decode=opentelemetry.proto.profiles.v1development.ProfilesData \
    --proto_path=. \
    opentelemetry/proto/profiles/v1development/profiles.proto \
    < profile.pb

# Ecosystem validation
profcheck profile.pb
```

Results:

- `protoc` → ✅  
- `profcheck` → ✅

The only remaining `profcheck` warnings were about timestamp ranges in synthetic test data, i.e. **not** protocol issues.

The mystery staircase of `attribute_indices` was gone. The logs looked calmer. Outside was still damp and miserable, but at least the protobuf wasn’t.

---

## 8. Dual Validation Setup (With Docker)

To avoid “works on my machine” in the future, we containerized the validation environment.

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
set -e

PROFILE_FILE="$1"

if [ -z "$PROFILE_FILE" ]; then
  echo "Usage: validate-profile <profile.pb>" >&2
  exit 1
fi

echo "=== protoc decode ==="
protoc --decode=opentelemetry.proto.profiles.v1development.ProfilesData \
    --proto_path=/proto/opentelemetry-proto \
    opentelemetry/proto/profiles/v1development/profiles.proto \
    < "$PROFILE_FILE" > /tmp/decoded.txt

echo "Decoded profile written to /tmp/decoded.txt"

echo
echo "=== profcheck ==="
profcheck "$PROFILE_FILE"
EOF

RUN chmod +x /usr/local/bin/validate-profile
```

With this image:

- Everyone in the team validates profiles with **the same**:
  - `profcheck` build
  - `protoc` version
  - OTLP proto checkout

CI uses it, local dev can use it, and nobody has to guess which version of which proto they’re really talking to.

---

## 9. Practical Takeaways

### 9.1. “Spec-Compliant” Needs a Commit Hash

It’s not enough to say:

> “We follow the OTLP profiles proto.”

You must also know:

- **Which commit** of that proto you follow
- Which commit your tools (Go modules, profcheck, agents, exporters) are generated from

Two schemas with the same package and message names but different field numbers are a silent disaster.

### 9.2. Protoc Is Necessary, Not Sufficient

`protoc` tells you:

> “This payload is valid for the proto you gave me.”

It does **not** guarantee:

- That this proto matches what `profcheck` was generated from
- That your Go/Java/Python modules are in sync with your `.proto` checkout

Think of `protoc` as the **local judge**, not the whole court.

### 9.3. Hex Dumps Still Matter

Hex dumps and manual decoding are tedious, but they:

- Prove which field numbers are actually on the wire
- Show whether you’re emitting packed vs. non-packed fields correctly
- Help you spot “incrementing values in the wrong field” patterns

When you’re stuck on a cold night, it’s basically looking for footprints in a snowstorm.

### 9.4. Stale Everything: Files *and* Schemas

Two equally annoying forms of “you’re staring at the wrong thing”:

- Old `.pb` artifacts from earlier builds
- Old proto versions baked into dependencies and tools

You have to invalidate both before you trust any conclusion.

### 9.5. Incrementing Values in a “Constant” Field = Schema Mismatch Alarm

If you expect:

- `attribute_indices = [1]` for all samples

but you see:

- `2, 3, 4, …, 101`

assume:

- You are probably decoding the wrong field under the wrong schema, not just “off by one.”

---

## 10. Debugging Checklist for Protobuf Weirdness

If you find yourself debugging protobuf in a winter mood, use this checklist:

1. **Verify internal data**
   - Log tables, sizes, indices, actual arrays
2. **Validate with `protoc`**
   - Against the `.proto` you *think* you’re implementing
3. **Inspect the wire**
   - Hex dump
   - Decode tags: `tag = (field_number << 3) | wire_type`
4. **Compare schemas**
   - GitHub trunk `.proto`
   - Vendored `.proto` in your repo
   - Generated code (Go/Java/etc.)
5. **Check tool versions**
   - Which commit is `profcheck` (or other validators) built from?
6. **Look at value patterns**
   - Incrementing sequences
   - Constant offsets
   - Suspicious repetition
7. **Regenerate everything**
   - Delete old `.pb` files
   - Clean & rebuild
8. **Assume version skew first**
   - Before blaming protobuf
   - Before blaming the cat
   - Before rewriting your encoder twice

---

## 11. Postmortem: What Actually Happened

- The OTLP profiles proto evolved; field numbers were rearranged in `Sample`
- The **Go module** used by `profcheck` was locked to an **older commit**
- We implemented using the **trunk** proto layout
- Our encoder wrote tags according to trunk
- `profcheck` decoded tags according to the older Go module layout
- Our `link_index` became `attribute_indices` in `profcheck`’s view
- `profcheck` legitimately complained about “index 101 out of range [0..2)”

The fix:

- Update four constants in `OtlpProtoFields.Sample` to match the Go module’s field numbering

The lesson:

- When two validators disagree on a cold, wet night, **suspect schema version skew first**
- And always verify which reality your tools are compiled against

The cat, for the record, was right to stay on the radiator the entire time.
