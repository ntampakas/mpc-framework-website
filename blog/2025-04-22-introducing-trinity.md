---
title: Introducing Trinity
description: Two-Party Computation for the ZK Era
slug: introducing-trinity
authors: yanis
date: 2025-04-22
tags: [trinity, hacking, 2pc, zk]
hide_table_of_contents: false
---

# Introducing Trinity: Two-Party Computation for the ZK Era

We’ve just added **Trinity** as a new template in [mpc-hello](https://github.com/voltrevo/mpc-hello/tree/main/trinity), and it’s time to show what it brings to the table.

<!--truncate-->

But first — what is **Trinity**, and why should you care?

### 🧠 A Classic 2PC Flow

Let’s start with a traditional secure two-party computation (2PC). Say Alice and Bob want to compute `a + b`, without revealing their private inputs `a` and `b` to each other.

This is typically done using:

- **Boolean circuits**: The computation is expressed as a sequence of logic gates (AND, OR, XOR, etc.). For example, adding two 8-bit numbers becomes a gate-level network, where each input bit is a wire. You can check out [Summon](https://github.com/voltrevo/summon/) for great examples.
- **Garbled Circuits (GC)**: One party (the _garbler_) encrypts the entire circuit. Each wire in the circuit gets two cryptographic labels (representing 0 or 1), and the circuit gates are encrypted so that only the correct output label can be unlocked, step by step. The other party (the _evaluator_) evaluates this encrypted circuit without ever seeing the real bits.
- **Oblivious Transfer (OT)**: The evaluator (Alice) needs to receive _just_ the labels corresponding to her input bits — but without revealing those bits to the garbler.

#### Classical OT Construction

In most traditional protocols, OT is built using **public key cryptography**.

Let’s say Alice has an input bit `a ∈ {0,1}`:

- Bob (the garbler) prepares two labels for Alice’s input wire: one for 0 and one for 1.
- Alice runs an OT protocol to retrieve only the label that corresponds to her bit, without revealing it.
- This typically involves public key operations where Bob encrypts both labels, and Alice can only decrypt one.

Together, these steps let two parties jointly compute a function without revealing their inputs.

### ✨ Enter Trinity

Trinity follows the same broad structure — but introduces key innovations to make the flow more modular, more verifiable, and better aligned with ZK tooling.

Specifically, Trinity leverages:

- 🔑 **Laconic OT**: Based on the [Witness Encryption for KZG](https://eprint.iacr.org/2024/264) paper, Trinity replaces traditional OT with a KZG-based commitment scheme.
- 📦 **KZG Commitments**:
  - Alice commits to her input using a polynomial commitment (KZG), without targeting Bob specifically. These commitments are reusable across sessions or peers.
  - Since some ZK provers (like [Halo2](https://github.com/privacy-scaling-explorations/halo2)) use KZG already, this means Alice can also **prove** her committed input in zero-knowledge — if she wants.

In short, Trinity lets you garble circuits in the usual way, but with reusable, verifiable inputs — perfect for privacy-preserving apps, on-chain protocols, or even MPC-enabled credentials.

Now let’s see how to build with it.

### Setup

We need to write our 2PC circuit first, thanks to [Summon](https://github.com/voltrevo/summon) is going to be super easy. Here we are going with a very simple add circuit, adding Alice and Bob numbers.

```ts
export default function main(a: number, b: number) {
  return a + b;
}
```

In the next snippet, we're going to initialise the trinity library and parse our circuit, so it can be consumed by both the garbler and the evaluator.

```ts
import * as summon from "summon-ts";
import getCircuitFiles from "./getCircuitFiles";
import { initTrinity, parseCircuit } from "@trinity-2pc/core";

export default async function generateProtocol() {
  await summon.init();
  const trinityModule = await initTrinity();

  const circuit = summon.compileBoolean(
    "circuit/main.ts",
    16,
    await getCircuitFiles()
  );

  const circuit_parsed = parseCircuit(circuit.circuit.bristol, 16, 16, 16);

  return { trinityModule, circuit_parsed };
}
```

### 🧬 Trinity in Action: Evaluator → Garbler → Evaluation

### Evaluator Commitment Phase

The first phase of the protocol requires the evaluator, here Alice, to commit to its input. For the sake of the example we're going to use the `Plain` setup (vs. Halo2, who's using full purpose ZK), performing a plain KZG commit.

```ts
// Call to the protocol generator we described above
const protocol = await generateProtocol();
// Create a Setup for our KZG protocol.
// Note: For a production setting, these keys should be generated and published.
// It should not be generated on the fly like in this example.
const trinitySetup = protocol.trinityModule.TrinityWasmSetup("Plain");
// Create an instance of a Trinity Evaluator
// and will generate the commitment to Alice's input.
// number: Alice's number a
const evaluator = protocol.trinityModule.TrinityEvaluator(
  trinitySetup,
  intToUint8Array2(number)
);
```

Now Alice can send both the parameters for the KZG setup and the commitment to her input.

```ts
const newSocket = await connect(code, "alice");

newSocket?.send(
  JSON.stringify({
    type: "setup",
    setupObj: Array.from(trinitySetup.to_sender_setup() || new Uint8Array()),
  })
);

newSocket?.send(
  JSON.stringify({
    type: "commitment",
    commitment: evaluator.commitment_serialized,
  })
);
```

### Garbler Phase

Bob has now received both the setup parameters and the commitment. He can now garble the circuit to encrypt his own input.

```ts
// Instantiate the Trinity wasm library
const protocol = await generateProtocol();
// Connect sockets
await connect(joiningCode, "bob");
// Get the messages from Alice
const setupMessage = (await msgQueue.shift()) as SetupMessage;
const setupObj = new Uint8Array(setupMessage.setupObj);
const commitmentMessage = (await msgQueue.shift()) as CommitmentMessage;
const commitment = commitmentMessage.commitment;
```

```ts
// Instantiate KZG setup from evaluator's setup
const garblerSetup = TrinityWasmSetup.from_sender_setup(setupObjValue);
// Instantiate a Trinity Garbler from the commitment and Bob's input
const garblerBundle = protocol?.trinityModule.TrinityGarbler(
  commitmentValue,
  garblerSetup,
  intToUint8Array2(number),
  protocol.circuit_parsed
);
const serializedBundle = Array.from(
  new Uint8Array(garblerBundle?.bundle || [])
);
// Send back the garbled data
socket?.send(
  JSON.stringify({
    type: "garblerBundle",
    garblerBundle: serializedBundle,
  })
);
```

### Evaluate the circuit

Now Alice can receive the garbeld data from the Bob and evaluate the circuit. Note that the process is assymetric, and only Alice evaluate the circuit on here side and can send back the result to Bob.

```ts
// Receiving serialized Garbled data
const bundleArray = new Uint8Array(parsedMsg.garblerBundle);
// Reconstructing Garbled data
const bundle = TrinityGarbler.from_bundle(bundleArray);
// Evaluate our circuit
const resultBytes = currentEvaluator.evaluate(
  bundle,
  currentProtocol?.circuit_parsed
);
// Get the result
const result = booleanArrayToInteger(resultBytes);
```

### 🚀 What’s Next?

This is a minimal example — but Trinity supports:

- Integration with ZK (via Halo2, Noir soon 👀)
- More complex circuits (via Summon)

Check out [mpc-hello](https://github.com/voltrevo/mpc-hello/tree/main/trinity) and [trinity](https://github.com/privacy-scaling-explorations/Trinity) to dive deeper, or come chat with us in [PSE Discord](https://discord.com/channels/943612659163602974/1319092543513690203)!
