<script lang="ts">
  import Button from "@smui/button";
  import Textfield from "@smui/textfield";
  import Card from "@smui/card";
  import Fab, { Icon, Label } from "@smui/fab";
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
      getPublicKey: () => Promise<string>;
      signEvent: (event: UnsignedEvent) => Promise<Event>;
    };
  }

  // constants
  // hash tag
  const signMessage = "from NaaS";

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

  async function onclick() {
    nip07 = !!(window as Window).nostr;

    // Generate key
    sk = nip07 ? "" : generatePrivateKey();
    pk = nip07
      ? await (window as Window).nostr.getPublicKey()
      : getPublicKey(sk);

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

    disabled = true;
    const message = `nostr:${yabumiNpub} dice ${diceNum}d${sideNum} \n${signMessage}`;

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
      ? await (window as Window).nostr.signEvent(unsignedEvent)
      : finishEvent(unsignedEvent, sk);

    // Request to bot
    pool.publish(relays, event);

    // cooling time
    // FIXME: We can request when update input.
    setTimeout(() => {
      disabled = false;
      // TODO: reuse pool.
      sub.unsub();
      pool.close(relays);
    }, 5000);
  }
</script>

<link
  href="https://fonts.googleapis.com/css2?family=Material+Icons&Roboto+Mono:ital@0;1&family=Roboto:ital,wght@0,100;0,300;0,400;0,500;0,700;0,900;1,100;1,300;1,400;1,500;1,700;1,900&display=swap"
  rel="stylesheet"
/>
<link
  rel="stylesheet"
  href="https://cdn.jsdelivr.net/npm/svelte-material-ui@6.0.0/bare.min.css"
/>
<main>
  <Fab extended style="height: 34px;font-size: 0.6em;width: auto;">
    <Icon class="material-icons" style="margin-right: 4px;">casino</Icon>
    <Label
      >823chan/Dice<a
        href={`https://nostter.vercel.app/${yabumiNpub}`}
        target="”_blank”"
        rel="noopener noreferrer"
        ><Icon
          class="material-icons"
          style="font-size: 1em; margin: 0 0 0 4px;width: 10px;vertical-align: middle;"
          >info</Icon
        ></a
      ></Label
    >
  </Fab>
  <div class="top-space" />
  <div class="input-flex-container">
    <div class="card-container"><Card padded>{reply}</Card></div>
  </div>

  <!-- dice number -->
  <div class="input-flex-container">
    <div class="text-input-container">
      <Textfield
        type="number"
        variant="outlined"
        bind:value={diceNum}
        label="Dice number"
        class="text-input"
      />
    </div>

    <!-- sides of the dice -->
    <div class="text-input-container">
      <Textfield
        type="number"
        variant="outlined"
        bind:value={sideNum}
        label="Sides number"
      />
    </div>
  </div>

  <Button variant="raised" on:click={onclick} {disabled}>Submit</Button>
  <p>
    サイコロの数：{diceNum ?? ""}
  </p>
  <p>
    サイコロの面の数：{sideNum ?? ""}
  </p>
  <div class="input-flex-container">
    <p class="message">
      pubkey: {pk}
    </p>
  </div>
</main>

<style>
  .top-space {
    margin-top: 8em;
  }
  @media screen and (max-height: 740px) {
    .top-space {
      margin-top: 2em;
    }
  }
  .card-container {
    margin: 4em;
    width: 100%;
    min-width: 200px;
    max-width: 400px;
    color: rgba(0, 0, 0, 0.5);
  }
  .input-flex-container {
    display: flex;
    justify-content: center;
    flex-wrap: wrap;
    width: 100%;
  }
  .text-input-container {
    margin: 0.5em;
    max-width: fit-content;
    min-width: 200px;
  }
  .message {
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    max-width: 300px;
  }
</style>
