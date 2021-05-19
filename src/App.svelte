<script lang="ts">
  import { onMount, onDestroy } from "svelte";
  import { TezosToolkit } from "@taquito/taquito";
  import { BeaconWallet } from "@taquito/beacon-wallet";
  import { NetworkType } from "@airgap/beacon-sdk";
  import { HubConnectionBuilder, HubConnection } from "@microsoft/signalr";

  let Tezos: TezosToolkit;
  let wallet: BeaconWallet;
  let subscription: HubConnection;
  let blockHead: { protocol: string; level: number; lastUpdate: string };
  let confirmed: { confirmation: string};
 
  const rpcUrl = "https://api.tez.ie/rpc/mainnet";
  const packages: { name: string; display: string; version: number }[] = [
    { name: "svelte", display: "Svelte", version: 3 },
    { name: "webpack", display: "Webpack", version: 5 },
    { name: "typescript", display: "TypeScript", version: 3 },
    { name: "taquito", display: "Taquito", version: 8 },
    { name: "beacon", display: "Beacon SDK", version: 2 }
  ];

  const connect = async () => {
    try {
      wallet = new BeaconWallet({
        name: "Send Tez to a Cat Wallet",
        preferredNetwork: NetworkType.MAINNET
      });
      await wallet.requestPermissions({
        network: {
          type: NetworkType.MAINNET,
          rpcUrl
        }
      });
      Tezos.setWalletProvider(wallet);
    } catch (err) {
      console.error(err);
    }
  };

  const disconnect = () => {
    wallet.client.destroy();
    wallet = undefined;
  };

  let success = false;
  let loading = false;
  const transfer = async () => {
    loading = true
    const amount = 0.1;
    const address = 'tz1bW6pZyZaotadceKT6KCRPG9MNb9cVMKo6';
    const op = await Tezos.wallet.transfer({ to: address, amount: amount }).send();
    await op.confirmation();
    success = true;
    loading = false
  }   


  const subscribeToEvents = async () => {
    const connection = new HubConnectionBuilder()
      .withUrl("https://api.tzkt.io/v1/events")
      .build();

    // auto-reconnect
    connection.onclose(subscribeToEvents);

    connection.on("head", msg => {
      blockHead = {
        protocol: msg.data.protocol,
        level: msg.data.level,
        lastUpdate: msg.data.timestamp
      };
    });

    // open connection
    await connection.start();
    // subscribe to head
    await connection.invoke("SubscribeToHead");
    // return connection instance
    return connection;
  };

  onMount(async () => {
    Tezos = new TezosToolkit(rpcUrl);
    const headerInfo = await Tezos.rpc.getBlockHeader();
    blockHead = {
      protocol: headerInfo.protocol,
      level: headerInfo.level,
      lastUpdate: headerInfo.timestamp
    };
    subscription = await subscribeToEvents();
  });

  onDestroy(async () => {
    // closes subscription when component unmounts
    await subscription.stop();
  });
</script>

<style lang="scss">
  $tezos-blue: #3b0303;

  .container {
    font-size: 20px;
    max-width: 80%;

    .subtitle {
      font-size: 30px;
      font-family: cursive;
      font-style: oblique;
      color: rgb(46, 5, 5);
      margin: 10px;
    }

     .chain-info {
       font-size: 20px;

       p {
         margin: 5px;
         font-family: cursive;
         font-style: oblique;
         color: $tezos-blue;
       }
     }

    button {
      appearance: none;
      border: solid 2px $tezos-blue;
      border-radius: 5px;
      background-color: white;
      padding: 20px;
      font-size: 20px;
      font-family: cursive;
      font-style: oblique;
      color: $tezos-blue;
      transition: 0.3s;
      cursor: pointer;
      outline: none;

      &:hover {
        color: white;
        background-color: $tezos-blue;
      }
    }
  }
</style>

<main>
  <div class="container">
    <div class="subtitle">Send Tez to a Cat</div>
    <br />
    <div>
      <img src={'images/buddy.jpg'} alt="A Cat" />
    </div>
    <br />
    <div>
      {#if wallet}
        <div>
          {#if loading}
            <img src={'images/spinning_arrows.gif'} alt="loading...">
            <div class="subtitle">    
              <p>The cat might be outside. Let's wait.</p>
            </div>
          {:else}
          <br />
            <button on:click={transfer}>Use this button to send the cat 0.1 Tez</button>  
            {#if success} 
            <br />
            <br />
            <button on:click={disconnect}>Close the wallet by using this button</button>
            <br />
            <div class="subtitle">   
             <p>Do you have treats?</p>
            </div>
            <br />
            
            <img src={'images/buddy-says-thanks.jpg'} alt="The cat says thanks!">
            <br />
            {/if}
          {/if} 
          </div>
      {:else}
        <button on:click={connect}>Use this button to open a wallet and connect a blockchain address</button>
        <br />
        <br />
      {/if}
    </div>   
    <br />
    <br />
    {#if blockHead}
      <div class="chain-info">
        <p>Protocol: {blockHead.protocol}</p>
        <p>Level: {blockHead.level}</p>
        <p>Block timestamp: {blockHead.lastUpdate}</p>
      </div>
    {/if}
    <br />
    <br />
  </div>
</main>
