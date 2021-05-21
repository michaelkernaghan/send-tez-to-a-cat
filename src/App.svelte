<script lang="ts">
  import { onMount } from "svelte";
  import { TezosToolkit } from "@taquito/taquito";
  import type { ContractAbstraction, Wallet } from "@taquito/taquito";
  import { BeaconWallet } from "@taquito/beacon-wallet";
  import { NetworkType } from "@airgap/beacon-sdk";

  type UserTickets =
    | { ticketer: string; value: string; amount: number }
    | undefined;

  let Tezos: TezosToolkit;
  let wallet: BeaconWallet;
  let userAddress = "";
  let userTickets: UserTickets = undefined;
  let userTicketValidity: string;
  let ticketer: ContractAbstraction<Wallet>;
  let ticketerStorage: any;
  let loadingProfile = true;
  let loadingBuy = false;
  let loadingRedeem = false;

  const rpcUrl = "https://api.tez.ie/rpc/florencenet";
  const ticketerAddress = "KT1Vv3kTDW4rz5JhgM8XZgxcrRwFhHXXQQt6";

  const fetchUserTickets = async (
    address: string,
    ticketerAddress: string
  ): Promise<{ tickets: UserTickets; validity: string }> => {
    let tickets: UserTickets;
    let ticketsValidity = "";
    try {
      ticketer = await Tezos.wallet.at(ticketerAddress);
      ticketerStorage = await ticketer.storage();
      const result = await ticketerStorage.tickets.get({
        0: address,
        1: "standard"
      });
      if (result) {
        tickets = result[1];
        ticketsValidity = result[0];
      } else {
        tickets = undefined;
        ticketsValidity = "";
      }
    } catch (error) {
      console.log(error);
    }

    return { tickets, validity: ticketsValidity };
  };

  const connect = async () => {
    try {
      wallet = new BeaconWallet({
        name: "Send Tez to a cat and get a ticket",
        preferredNetwork: NetworkType.FLORENCENET
      });
      await wallet.requestPermissions({
        network: {
          type: NetworkType.FLORENCENET,
          rpcUrl
        }
      });
      Tezos.setWalletProvider(wallet);
      userAddress = await wallet.getPKH();
      const { tickets, validity } = await fetchUserTickets(
        userAddress,
        ticketerAddress
      );
      userTickets = tickets;
      userTicketValidity = validity;
    } catch (err) {
      console.error(err);
    } finally {
      loadingProfile = false;
    }
  };

  const disconnect = () => {
    wallet.client.destroy();
    wallet = undefined;
    userAddress = "";
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

  const buyTickets = async (ticketAmount: number) => {
    loadingBuy = true;
    try {
      const pricePerTicket = await ticketerStorage.data.valid_ticket_types.get(
        "standard"
      );
      const op = await ticketer.methods
        .buy_tickets(ticketAmount, userAddress, "standard")
        .send({
          amount: pricePerTicket.toNumber() * ticketAmount,
          mutez: true
        });
      await op.confirmation();
      // refreshes the storage
      ticketerStorage = await ticketer.storage();
      // gets user's tickets
      const result = await ticketerStorage.tickets.get({
        0: userAddress,
        1: "standard"
      });
      if (result) {
        userTickets = result[1];
        userTicketValidity = result[0];
      } else {
        userTickets = undefined;
        userTicketValidity = "";
      }
    } catch (error) {
      console.log(error);
    } finally {
      loadingBuy = false;
    }
  };

  const redeemTicket = async () => {
    loadingRedeem = true;
    try {
      const op = await ticketer.methods.redeem_ticket("standard").send();
      await op.confirmation();
      // refreshes the storage
      ticketerStorage = await ticketer.storage();
      // gets user's tickets
      const result = await ticketerStorage.tickets.get({
        0: userAddress,
        1: "standard"
      });
      if (result) {
        userTickets = result[1];
        userTicketValidity = result[0];
      } else {
        userTickets = undefined;
        userTicketValidity = "";
      }
    } catch (error) {
      console.log(error);
    } finally {
      loadingRedeem = false;
    }
  };

  onMount(async () => {
    Tezos = new TezosToolkit(rpcUrl);
    wallet = new BeaconWallet({
      name: "Send Tez to a cat and get a ticket",
      preferredNetwork: NetworkType.FLORENCENET
    });
    const activeAccount = await wallet.client.getActiveAccount();
    if (activeAccount) {
      Tezos.setWalletProvider(wallet);
      userAddress = activeAccount.address;
      const { tickets, validity } = await fetchUserTickets(
        userAddress,
        ticketerAddress
      );
      userTickets = tickets;
      userTicketValidity = validity;
      loadingProfile = false;
    }
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

    .tickets {
       font-size: 20px;
       font-family: cursive;
       font-style: oblique;
       color: $tezos-blue;
       
       p {
         margin: 20px;
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

  .loading {
    background: linear-gradient(92deg, #dce8f9, #2c0202);
    background-size: 400% 400%;
    color: white !important;
    border: solid 2px white !important;
    -webkit-animation: loading 2s ease infinite;
    -moz-animation: loading 2s ease infinite;
    animation: loading 2s ease infinite;
  }

  @-webkit-keyframes loading {
    0% {
      background-position: 0% 57%;
    }
    50% {
      background-position: 100% 44%;
    }
    100% {
      background-position: 0% 57%;
    }
  }
  @-moz-keyframes loading {
    0% {
      background-position: 0% 57%;
    }
    50% {
      background-position: 100% 44%;
    }
    100% {
      background-position: 0% 57%;
    }
  }
  @keyframes loading {
    0% {
      background-position: 0% 57%;
    }
    50% {
      background-position: 100% 44%;
    }
    100% {
      background-position: 0% 57%;
    }
  }
</style>

<main>
  <div class="container">
    <div class="subtitle">Send Tez to a cat and get a ticket</div>
    <br />
    <div><img src={'images/buddy.jpg'} alt="A Cat" /></div>
    <br />
    <br />
    <div>
      {#if userAddress}
        {#if loadingProfile}
          <div class="tickets"></div>
          <img src={'images/CatlickingPaw.gif'} alt="loading...">
        {:else}
          <div class="tickets">
            {#if userTickets && userTicketValidity}
              <div>Cat tickets you have: {userTickets.amount}</div>
              <!-- <div>Ticket type: {userTickets.value}</div> -->
              <div>Cat Ticketer: {ticketerAddress}</div>
              <div>
                Valid until: {new Date(
                  +new Date(userTicketValidity).getTime() +
                    +ticketerStorage.data.ticket_validity * 1000
                ).toISOString()}
              </div>
            {:else}
              <p>You should get a cat ticket.</p>
            {/if}
          </div>
          <br />
          <div class="buttons">
            <button class:loading={loadingBuy} on:click={async () => await buyTickets(1)}>Buy 1 Cat Ticket</button>
            {#if userTickets && userTickets.amount > 0}
              <button class:loading={loadingRedeem} on:click={redeemTicket}>
                Redeem 1 Cat Ticket
              </button>
            {/if}
            {#if loading}
            <br/>
            <br/>
            <img src={'images/CatlickingPaw.gif'} alt="loading...">
            {:else}
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
        {/if}
      {:else}
        <button on:click={connect}>Send the cat 0.1 Tez</button>
      {/if}
    </div>
  </div>
</main>
