<script lang="ts">
    import {onMount} from "svelte";
    import { ethers } from "ethers";
    import {asyncForEach, sleep} from "../utils/Utils.svelte";

    // interfaces
    interface keyable {
        [key: string]: any  
    }

    // const
    const N = `
`;

    // blockchain
    let provider;
    let wallet;

    // transactions
    let signed_transactions:string = "";
    let tx_result_map:keyable = {};

    // reactivity
    $: calculateNumSignedTransactions(signed_transactions);

    // counts
    let num_signed_transactions = 0;

    // loading
    let loading = false;


    const setupProvider = () => {
        // get provider
        provider = new ethers.providers.JsonRpcProvider("https://proxy.roninchain.com/free-gas-rpc", 2020);
        //provider = new ethers.providers.JsonRpcProvider("https://api.roninchain.com/rpc", 2021);

        // get wallet from mnemonic
    }


    onMount(() => {
        setupProvider();
    });

    
    /**
     * Reactivity
     */
     const calculateNumSignedTransactions = (_signed_transactions:string) => {
        num_signed_transactions = _signed_transactions.split(N+N).filter(x => x).length || 0;
    }


    /**
     * Sets or updates a specific transaction at a index in the map with new a status and data  
     * @param _index
     * @param _type
     * @param _status
     * @param _data
     */
    const setTxResultOnMap = (_index:number, _type:string, _status:string, _data:any) => {

        console.log("tx:info ", _index, _type, _status, _data);

        tx_result_map[_index] = {
            type: _type,
            status: _status,
            data: _data
        };
    }

    /**
     * Broadcasts multiple transactions to the ronin network
     */
    const broadcastTransactions = async () => {

        loading = true;

        let txs = signed_transactions.split(N+N).filter(x => x);

        console.log("txs", txs);

        await sleep(750);

        await asyncForEach(txs, async (signed_tx, i) => {

            await sleep(80);

            let tx_result = null;
            try {
                setTxResultOnMap(i, "success", "tx_created", null);
                // created tx
                tx_result = await provider.sendTransaction(signed_tx);
            }
            catch(err){
                // create tx error
                setTxResultOnMap(i, "error", "tx_send_error", err);
            }

            let tx_confirmed = null;
            if(tx_result) {
                try {
                    setTxResultOnMap(i, "success", "tx_sent", tx_result);
                    // confirmed tx
                    tx_confirmed = await tx_result.wait();
                }
                catch(err){
                    // confirm tx error
                    setTxResultOnMap(i, "error", "tx_confirm_error", err);
                }
            }

            if(tx_confirmed) {
                setTxResultOnMap(i, "success", "tx_confirmed", tx_confirmed);
            }
        });

        loading = false;
    }

</script>


<div class="area">

    <div class="titlebar">
        <div class="sub">Ronin Transaction Broadcaster</div>
        <h1 class="b">Ronin TX Broadcaster</h1>
    </div>

    <div class="box">

        <div class="input">
            <div class="line">
                <div class="label">
                    Signed Transactions:
                    {#if num_signed_transactions > 0}
                        <span class="counter">({num_signed_transactions}X)</span>
                    {/if}
                </div>
                <textarea type="text" bind:value={signed_transactions} />
            </div>
            <div class="center">
                <button on:click={broadcastTransactions}>
                    Broadcast Transactions
                </button>
            </div>
        </div>

        <div class="output">
            {#if loading}
                <div class="loader">
                    loading...
                </div>

            {/if}

            <div>
                <div class="title2">Transactions ({Object.values(tx_result_map).length})</div>
                {#each Object.values(tx_result_map) as tx,i}
                    <div class="tx">
                        <div style="width:50px; margin-right:20px;">#{i + 1}</div>
                        <div style="width:80px; margin-right:20px;"><span class={`type_tag ${tx.type}`}>{tx.type}</span></div>
                        <div style="width:100px; margin-right:20px;"><span class={`status_tag ${tx.status}`}>{tx.status}</span></div>
                        <div style="width:100px; margin-right:20px;">

                            {#if tx.status == "tx_sent" && tx?.data?.hash}
                                <a target="_blank" href={`https://explorer.roninchain.com/tx/${tx?.data?.hash}`}>TX Link (unconfirmed)</a>
                            {/if}

                            {#if tx.status == "tx_confirmed" && tx?.data?.transactionHash}
                                <a target="_blank" href={`https://explorer.roninchain.com/tx/${tx?.data?.transactionHash}`}>TX Link</a>
                            {/if}

                        </div>
                        <div class="data_field">
                            {JSON.stringify(tx.data)}
                        </div>
                    </div>
                {/each}
            </div>
        </div>
    </div>
</div>


<style>

    .titlebar {
        display:flex;
        flex-flow:column;
    }

    .titlebar .sub {
        font-size: 16px;
        margin-bottom: 0;
        color: #b7b7b7;
        font-family: monospace;
    }

    .status_tag {
        font-size:14px;
        background: #333333;
        color: white;
        border-radius: 50px;
        padding: 2px 15px;
        display: inline-flex;
        font-size: 12px;
    }
    .status_tag.tx_sent {
        background:#dba121;
    }
    .status_tag.tx_confirmed {
        background:#4caf50;
    }

    .type_tag {
        background: #333333;
        color: white;
        border-radius: 50px;
        padding: 2px 15px;
        display: inline-flex;
        font-size: 12px;
    }

    .type_tag.error{
        background:#cf1a65;
    }
    .type_tag.success{
        background:#4caf50;
    }

    .txs {
        display:flex;
        flex-flow:column;
    }

    .tx {
        display:flex;
        padding: 8px 2px;
        align-items: center;
    }
    .tx:hover {
        background: #f3f3f3;
    }

    .data_field {
        font-size:12px;
        max-width: 500px;
        overflow-x:auto;
    }

    .title2 {
        font-size: 24px;
        margin-bottom: 25px;
    }

    .loader {
        color: grey;
    text-align: center;
    margin: 10px 0;
    }

    .line {
        margin-bottom:10px;
    }
    .center {
        display:flex;
        align-items:center;
        justify-content: center;
    }

    .label {
        margin-bottom: 8px;
        font-weight: 400;
        font-size: 14px;
        color: #007de7;
    }

    .output {
        margin-top:40px;
    }

    .counter {
        font-weight:bold;
    }

    button {
        background: #3192f5;
        color: white;
        border: none;
        border-radius: 50px;
        padding: 0 35px;
        height: 35px;
        cursor:pointer;
        margin-top:10px;
    }
    button:hover {
        background:#55a9ff;
    }

    h1 {
        margin:0;
        margin-bottom: 40px;
    }

    h1 .sub {
        font-weight:normal;
    }

    .area {
        padding:20px 25px;
        margin:0 auto;
        max-width:1200px;
        background: #ffffff;
        border-radius: 14px;
        box-shadow: 0 5px 18px #00000021;
    }

    input {
        width:100%;
        font-size:14px;
    }
    .seed_input {
        width: 100%;
    }

    textarea {
        font-size: 11px;
        font-family: monospace;
        font-weight: 400;
        line-height: 130%;

        width:100%;
        height:60px;
        border-color: #7c7c7c;
        /*background: #fafafa;*/
        resize: vertical;
    }

    .signed_tx_output {
        background: #fafafa;
    }
</style>