<template>
    <div class="m-10">
        <div class="flex flex-row">
            <div class="pr-2"><button @click="login('arconnect')" class="btn mt-2" :disabled="loggedIn">Login With ArConnect</button></div>
            <div class="pr-2"><button @click="login('othent')" class="btn mt-2" :disabled="loggedIn">Login With Othent</button></div>
            <div v-if="loggedIn"><button @click="logout" class="btn mt-2">Logout</button></div>
        </div>
        <div v-if="loggedIn">
            <div>
                Wallet ID: {{ walletAddress }} <br/>
                Wallet Balance: {{ walletBalance / 1000000 }} <br/>
                <span v-if="walletBalance == 0">Ask the AFTR team for some money!</span><br/>
            </div>
            <div>
                Contract ID: {{ CURRENCY_ID }} <br/>
                <button @click="onFaucet" class="btn btn-primary mt-2" :disabled="!loggedIn">Run...</button>
            </div>
            
        </div>

        <span v-if="loading" class="loading loading-dots loading-lg"></span>


        <div v-if="loggedIn" class="grid grid-cols-2 gap-4 mt-4 p-4 border">
            <div class="pt-4 w-full">
                <p class="font-sans font-lg text-aftrBlue">Input</p><br/>
                <vue-json-pretty :path="'res'" :data="input" :showDoubleQuotes="false" :deep=3 :deepCollapseChildren="false" :showLength="true" :showSelectController="true"> </vue-json-pretty>
            </div>
            <div class="pt-4 w-full">
                <p class="font-sans font-lg text-aftrBlue">Read Output</p><br/>
                <vue-json-pretty :path="'res'" :data="contractState" :showDoubleQuotes="false" :deep=3 :deepCollapseChildren="false" :showLength="true" :showSelectController="true" :collapsedOnClickBrackets="true" :showKeyValueSpace="true"> </vue-json-pretty>
            </div>
        </div>

        <div v-if="loggedIn">
            <button @click="signTx" class="btn btn-secondary mt-2" :disabled="!loggedIn">Sign...</button>
            <span v-if="loading" class="loading loading-dots loading-lg">Signing...</span>
            <p v-if="executionTime > 0">Execution time: {{ executionTime }} milliseconds</p>
        </div>
    </div>
</template>

<script>
import Arweave from "arweave";
import VueJsonPretty from 'vue-json-pretty';
import 'vue-json-pretty/lib/styles.css';
import { warpRead, warpWrite } from "./utils/warpUtils.js";
import othent from "./utils/othent";
import arconnect from "./utils/arconnect";
import * as arweaveWallet from "@othent/kms-aftr";
import { WarpFactory, defaultCacheOptions } from "warp-contracts";
import { DeployPlugin } from "warp-contracts-plugin-deploy";
import { InjectedArweaveSigner } from "warp-contracts-plugin-signature";

export default {
    components: { VueJsonPretty },
    data() {
        return {
            CURRENCY_ID: "XIWQaJ5F7FNgkoNso30bvZ0WvRyDWCs7DeG6Y_ZzDGI",
            input: { function: 'runFaucet' },
            executionTime: 0,
            contractState: {},
            user: {},
            loading: false,
            loggedIn: false,
            network: "mainnet",
            provider: "",
            walletAddress: "",
            walletBalance: 0,
            workContract: {},
        };
    },
    computed: {

    },
    methods: {
        async signTx() {
            const startTime = performance.now();
            this.loading = true;
            const arweave = new Arweave({
                    host: "ar-io.net",
                    port: 443,
                    protocol: "https"
            });
            // create a transaction
            let transaction = await arweave.createTransaction({
                data: '<html><head><meta charset="UTF-8"><title>Hello permanent world! This was signed via ArConnect!!!</title></head><body></body></html>'
            });
            if (this.provider === "arconnect") {
                // connect to the extension
                await window.arweaveWallet.connect(["SIGN_TRANSACTION"]);

                // sign using arweave-js
                const signedFields = await window.arweaveWallet.sign(transaction);

                // update transaction fields with the
                // signed transaction's fields
                transaction.setSignature({
                    id: signedFields.id,
                    owner: signedFields.owner,
                    reward: signedFields.reward,
                    tags: signedFields.tags,
                    signature: signedFields.signature
                });
            } else if (this.provider === "othent") {
                await arweaveWallet.sign(transaction);
                // await arweaveWallet.post(transaction);
            }
            const endTime = performance.now();
            this.executionTime = endTime - startTime;
            this.loading = false;
        },
		async onFaucet() {
            this.loading = true;
            this.contractState = {};
			const input = { function: "mintFaucet" };
			// let tx = await warpWrite(this.CURRENCY_ID, input, true, true, "PROD");
            let tx = await this.warpInteractWrite(this.CURRENCY_ID, input);
            this.contractState = await warpRead(this.CURRENCY_ID, true, "PROD");
            this.loading = false;
		},
        async login(provider = "arconnect") {
            this.loading = true;
			try {
				switch (provider) {
					case "othent":
						this.user = await othent.login();
						break;
					case "arconnect":
						this.user = await arconnect.connect();
						break;
					default:
						throw new Error("Invalid provider: " + provider);
				}
				this.provider = provider;
                this.loggedIn = true;
			} catch (e) {
                this.loggedIn = false;
				console.error(e);
                this.loading = false;
                return;
			}
            this.walletAddress = this.user.contractId;

            await this.getWorkBalances();

            this.loading = false;
		},
		async logout() {
			if (this.loggedIn) {
				if (this.provider == "othent") {
					await othent.logout();
				} else if (this.provider == "arconnect") {
					await arconnect.disconnect();
				}
                this.walletAddress = "";
                this.walletBalance = 0;
                this.provider = "";
                this.contractState = {};
                this.loading = false;
                this.loggedIn = false;
				console.info("Logged out successfully");
			} else {
				console.error("User must be logged in to log out");
			}
		},

        async getWorkBalances() {
            this.workContract = await warpRead(this.CURRENCY_ID, true, "PROD");
            if (this.workContract.balances.hasOwnProperty(this.walletAddress)) {
                this.walletBalance = this.workContract.balances[this.walletAddress];
            }
        },
        async warpInteractWrite(contractId, input) {
            const warp = WarpFactory.forMainnet({ ...defaultCacheOptions }).use(new DeployPlugin());
            // const warp = WarpFactory.forMainnet({ ...defaultCacheOptions, inMemory: true }).use(new DeployPlugin());
            // const warp = WarpFactory.forMainnet({ ...defaultCacheOptions }).use(new DeployPlugin());

            let signer;
            try {
                if (this.provider == "othent") {
                    signer = new InjectedArweaveSigner(arweaveWallet);
                    signer.getAddress = arweaveWallet.getActiveKey;
                } else if (this.provider == "arconnect") {
                    signer = new InjectedArweaveSigner(window.arweaveWallet);
                    signer.getAddress = window.arweaveWallet.getActiveAddress;
                }
                await signer.setPublicKey();
                const contract = warp
                    .contract(contractId)
                    .setEvaluationOptions({
                        internalWrites: true,
                        remoteStateSyncEnabled: true,
                        // remoteStateSyncSource: "https://dre-3.warp.cc/contract",
                        // remoteStateSyncSource: process.env.DRE_ENDPOINT,
                    })
                    .connect(signer);

                const { originalTxId } = await contract.writeInteraction(input);
                await warp.close();
                return originalTxId;
            } catch (err) {
                console.log(err);
                throw err;
            }
        },
    },
}
</script>