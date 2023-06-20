<script lang="ts">
  import { onMount, onDestroy } from "svelte";
  import Button from "@smui/button";
  import Textfield from "@smui/textfield";
  import Card from "@smui/card";
  import {
    finishEvent,
    generatePrivateKey,
    getPublicKey,
    SimplePool,
    type Event,
    type UnsignedEvent,
    type Sub,
  } from "nostr-tools";

  // nip07 types
  interface Window {
    nostr?: {
      getPublicKey: () => string;
      signEvent: (event: UnsignedEvent) => Event;
    };
  }

  // constants
  // yabumi chan id
  const yabumiHexPub =
    "3aa38bf663b6c834a04a6542edf14a81d3223e050c3cc9b7479f8c869c432cf2";
  const yabumiNpub =
    "npub1823chanrkmyrfgz2v4pwmu22s8fjy0s9ps7vnd68n7xgd8zr9neqlc2e5r";

  // Dice
  let diceNum = null;
  let sideNum = null;

    // button
  $: disabled = !diceNum || !sideNum; 

  // Bot reply
  let reply = "...waiting";

  // Your credential data
  let pk = "";
  let sk = "";
  let nip07 = false;

  // Setup relay
  const pool = new SimplePool();
  let relays = [
    "wss://relay.damus.io",
    "wss://relay-jp.nostr.wirednet.jp",
    "wss://nos.lol",
    "wss://yabu.me",
    "wss://holybea.com",
  ];
  let sub: Sub;

  onMount(() => {
    nip07 = !!(window as Window).nostr;

    // Generate key
    sk = nip07 ? "" : generatePrivateKey();
    pk = nip07 ? (window as Window).nostr.getPublicKey() : getPublicKey(sk);

    // in onClick
    sub = pool.sub(relays, [
      {
        kinds: [1],
        authors: [yabumiHexPub],
        "#p": [pk],
      },
    ]);

    sub.on("event", (event) => {
      reply = event.content;
    });
  });

  onDestroy(() => {
    sub.unsub();
    pool.close(relays);
  });

  function onclick() {
    disabled = true;
    const message = `nostr:${yabumiNpub} dice ${diceNum}d${sideNum}`;

    /**
     * Prepare event.
     */
    const unsignedEvent: UnsignedEvent = {
      kind: 1,
      created_at: Math.floor(Date.now() / 1000),
      tags: [["p", yabumiHexPub]],
      content: message,
      pubkey: pk,
    };

    const event = nip07
      ? (window as Window).nostr.signEvent(unsignedEvent)
      : finishEvent(unsignedEvent, sk);

    // Request to bot
    pool.publish(relays, event);

    // cooling time
    // FIXME: We can request when update input.
    setTimeout(() => disabled=false, 5000);
  }
</script>

<link
  rel="stylesheet"
  href="https://cdn.jsdelivr.net/npm/svelte-material-ui@6.0.0/bare.min.css"
/>
<main>
  <div class="card-container message"><Card class="message-card" padded>{reply}</Card></div>
  <!-- dice number -->
  <Textfield
    type="number"
    variant="outlined"
    bind:value={diceNum}
    label="Dice number"
  />
  <!-- sides of the dice -->
  <Textfield
    type="number"
    variant="outlined"
    bind:value={sideNum}
    label="Sides number"
  />
  <Button variant="raised" on:click={onclick} {disabled}>
    Submit
  </Button>
  <p>
    サイコロの数：{diceNum ?? ""}
  </p>
  <p>
    サイコロの面の数：{sideNum ?? ""}
  </p>
  <p class="message">
    pubkey: {pk}
  </p>
</main>

<style>
  .card-container {
    margin: 3em;
  }
  .message {
    color: rgba(0, 0, 0, .5);
  }
</style>
