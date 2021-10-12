
These are the notes I took while trying to figure out how Solana's `spl-governace` program works. My advice is to actually go through these steps-- once you complete them all you will have a much better understanding of everything.

## Useful links
* A [list of instructions][] supported by spl-governance
* A list of [implementation primitives][] that perform governance actions

[list of instructions]: https://docs.rs/spl-governance/1.1.1/spl_governance/instruction/index.html
[implementation primitives]: https://github.com/solana-labs/solana-program-library/tree/master/governance/program/src/processor

## Setup

* Install Rust and Solana CLI tools (the Anchor project has good [installation instructions][]
* Start a local solana cluster
  * Run `solana-node-validator -r` (the `-r` flag resets any previous state)
* Setup and fund a localhost wallet
  * Instal the Phantom wallet chrome extension and go through setup instructions
  * In the Phantom wallet click on the little gear icon, click "Change Network", and point the wallet to `localhost`
  * In the Phantom wallet click on the little gear icon and click on "Show Secret Recovery Phrase". Copy it.
  * Run `solana-keygen recover 'prompt://?key=0/0'` and paste the recovery phrase. This will import the phantom wallet private key into Solana CLI. (Note that the wallet and CLI use different default derivation paths, so the Phantom's `0/0` path must be specified to Solana CLI explicitly.)
  * Run `solana airdrop 1000` to fund the wallet. If you run `solana balance` you should now see 1000 SOL. You should also see 1000 SOL in your Phantom wallet.
* Deploy the spl-governance program
  * Run `git clone https://github.com/solana-labs/solana-program-library.git`. This will pull the solana program library codebase which, among other things, contains the governance program.
  * `cd` into `solana-program-library` and run `cargo build-bpf`. This will build all the programs (if you figure out how to build just `governance` let me know)
  * Run `solana program deploy target/deploy/spl_governance.so` to deploy the governance program to your local cluster. Note the printed Program Id.
* Run the Oyster governance GUI
  * Run `git clone https://github.com/solana-labs/oyster.git`. This will pull the web Ui for the governance program (among other things)
  * `cd` into `oyster` and run `yarn bootstrap` followed by `yarn start governance`. This will start the web ui.
  
[installation instructions]: https://project-serum.github.io/anchor/getting-started/installation.html

## Creating a Realm

A "realm" is an account that ties everything in `spl-governance` together. A realm is basically a DAO, so you create one realm account per DAO. So the first step is to create a realm account.

* Open the webui and add `?programId=GOVERNANCE_PROGRAM_ID_HERE` to the query string in the url bar to point the ui to your deployed governance program.
* Click "Connect" on the top right of the GUI to connect it to your Phantom wallet
* Click "Register realm"
* Pick any name you want
* Realms support "community" and "council" tokens. You can then push any proposal either through community voting or council voting. One way to think about it is that the "council" can act like admins of the DAO that can override or bypass the will of the community. For brewity we're only going to create community tokens here. The process for creating council tokens is the same.
* Create a community token
  * Run `spl-token create-token`. Note the token mint address.
  * Paste the token mint address into the "community token mint" input box
  * In the UI click "Register" to register the realm
  * Other options
	* "min community tokens to create governance"-- percent of the token supply you need to create governances. This can be used to prevent small shareholders to spam governances.
	* "advance settings" > "community mint supply factor (max vote weight)"-- fraction of the minted token to use as max vote weight. For example if there are 100 tokens minted and the fraction is `.5` (i.e. `50%`), it would take `26` tokens for the vote to pass assuming `51%` majority is required (`100 tokens * .5 / 2 + 1`). This setting is documented here-- https://docs.rs/spl-governance/1.1.1/spl_governance/state/enums/enum.MintMaxVoteWeightSource.html
* Relevant resources
  * This runs the program command `create_realm` documented here-- https://docs.rs/spl-governance/1.1.1/spl_governance/instruction/fn.create_realm.html
  * Relevant ts code is at [https://github.com/solana-labs/oyster/blob/main/packages/governance/src/views/home/registerRealmButton.tsx](https://github.com/solana-labs/oyster/blob/main/packages/governance/src/views/home/registerRealmButton.tsx) (see call to `registerRealm` on line 77)

## Creating a governance

A "governance" is an account for collectively controlling an asset. You create a governance per asset you want the DAO to control. For example if you wanted the DAO to control minting new supply of a particular token, you'd create a governance to manage that mint. If you wanted the DAO to control transfering a token out of a treasury's token account, you'd create another governance for that treasury. You'd have as many governance accounts as you have things you want the DAO to control.

### Step 1: prerequisites

* To create a governance you need to hold a minimum amount of a community governance token (we specified this during realm creation). So a prerequisite to creating a governance is minting some amount of governance token into an account you control.
  * Run `spl-token create-account TOKEN_MINT_ADDRESS_HERE` to create an account to hold the token. Note the account address. The token mint address is address for the community token mint we've created before.
  * Run `spl-token mint TOKEN_MINT_ADDRESS_HERE 100` to mint 100 units of the token into the holding account
  * In the realm UI the "Deposit Governance Tokens" button will become enabled. Before you can create a governance or vote, you need to deposit your governance tokens. (The governance tokens you simply hold in your account can't be used to perform realm operations).
	* Click the "deposit governance tokens" button and confirm the deposit. (You can always withdraw your tokens)
* Now if you click the three dots in the realm UI "Create New Governance" will be enabled
* Relevant resources
  * TS code is at [https://github.com/solana-labs/oyster/blob/226ab647144f4a9c7739104b82879dc8c136c7b3/packages/governance/src/views/realm/buttons/depositGoverningTokensButton.tsx#L55](https://github.com/solana-labs/oyster/blob/226ab647144f4a9c7739104b82879dc8c136c7b3/packages/governance/src/views/realm/buttons/depositGoverningTokensButton.tsx#L55)
  * Instruction is documented here [https://docs.rs/spl-governance/1.1.1/spl_governance/instruction/fn.deposit_governing_tokens.html]()

### Step 2: registering a governance

There are four types of governance you can create: program, mint, token account, and account. The first three are useful for governing program version upgrades, mint supply of a token, and transfering token out of an account, respectively. I think the "account" governance is a less specialized governance for executing arbitrary code on accounts in general, but I haven't verified this yet.

* Click "Create New Governance"
* Let's create a mint governance for our community token. What that means is that the DAO will be in control of further community governance token supply.
* Click "Mint" in the tab bar in the dialog
  * For mint address input the token mint address we've created before
  * Check "transfer authority" checkbox. If you don't do this, the command will create the governance object, but in order for it to govern you'd have to transfer authority over the mint to it manually.
  * Pick zero for "min instruction hold up time (days)". Otherwise you'll have to wait a day before you can execute a proposal after it passes.
  * Click "create"
* Relevant resources
  * ts code here-- [https://github.com/solana-labs/oyster/blob/226ab647144f4a9c7739104b82879dc8c136c7b3/packages/governance/src/views/realm/buttons/registerGovernanceButton.tsx#L82](https://github.com/solana-labs/oyster/blob/226ab647144f4a9c7739104b82879dc8c136c7b3/packages/governance/src/views/realm/buttons/registerGovernanceButton.tsx#L82)
  * rust instruction docs here-- [https://docs.rs/spl-governance/1.1.1/spl_governance/instruction/fn.create_mint_governance.html](https://docs.rs/spl-governance/1.1.1/spl_governance/instruction/fn.create_mint_governance.html)
  * Docs for various governance config fields-- [https://docs.rs/spl-governance/1.1.1/spl_governance/state/governance/struct.GovernanceConfig.html](https://docs.rs/spl-governance/1.1.1/spl_governance/state/governance/struct.GovernanceConfig.html). Surprising, but the docs here actually do a good job explaining what various parameters do.

## Creating a proposal

### Step 1: create a proposal draft

* In the governance screen click "Add new proposal"
* Enter a name (e.g. "Double token supply")
* Click "Add proposal"
* Relevant resources
  * ts code here-- [https://github.com/solana-labs/oyster/blob/226ab647144f4a9c7739104b82879dc8c136c7b3/packages/governance/src/views/governance/buttons/newProposalButton.tsx#L93](https://github.com/solana-labs/oyster/blob/226ab647144f4a9c7739104b82879dc8c136c7b3/packages/governance/src/views/governance/buttons/newProposalButton.tsx#L93)
  * instruction docs here-- [https://docs.rs/spl-governance/1.1.1/spl_governance/instruction/fn.create_proposal.html](https://docs.rs/spl-governance/1.1.1/spl_governance/instruction/fn.create_proposal.html)

### Step 2: add instructions to the proposal

Now that we have the draft of a proposal, we can add instructions to it. These are serialized Solana instructions that we can execute if the proposal were to pass.

* Click the little plus icon under "New Instruction"
* Let's add an instruction to mint more tokens. For the destination address paste your token account address (this is the account you created before to hold the token). Pick `100` for the amount.
* Hit "Create". (Note: this only adds the instruction on the client, not yet on chain. Also, if you hit the little plus icon and create an instruction again, it will overwrite the one you just added. The UI is super confusing here)
* Hit the little floppy icon. This will add the instruction you've stored in the browser to the proposal on chain.
  * Note: you can repeat this process and add more instructions, but the UI won't show the ones you've added to chain unless you refresh the page and reconnect to the wallet. So unless you refresh you won't see the instructions you've already added.
* Relevant resources
  * ts code where minting instruction gets encoded-- [https://github.com/solana-labs/oyster/blob/226ab647144f4a9c7739104b82879dc8c136c7b3/packages/governance/src/views/proposal/components/instructionInput/splTokenMintToForm.tsx#L52](https://github.com/solana-labs/oyster/blob/226ab647144f4a9c7739104b82879dc8c136c7b3/packages/governance/src/views/proposal/components/instructionInput/splTokenMintToForm.tsx#L52)
  * ts code that adds instruction to the proposal on chain (same for all instructions)-- [https://github.com/solana-labs/oyster/blob/226ab647144f4a9c7739104b82879dc8c136c7b3/packages/governance/src/views/proposal/components/instruction/newInstructionCard.tsx#L57](https://github.com/solana-labs/oyster/blob/226ab647144f4a9c7739104b82879dc8c136c7b3/packages/governance/src/views/proposal/components/instruction/newInstructionCard.tsx#L57)
  * Rust instruction docs-- [https://docs.rs/spl-governance/1.1.1/spl_governance/instruction/fn.insert_instruction.html](https://docs.rs/spl-governance/1.1.1/spl_governance/instruction/fn.insert_instruction.html)
  
### Step 3: sign off on the proposal

* Before anyone can vote on the proposal, you have to sign off on it. To do that click the blue "Sign Off" button on the top right.
* Relevant resources
  * ts code that calls sign off on chain: [https://github.com/solana-labs/oyster/blob/226ab647144f4a9c7739104b82879dc8c136c7b3/packages/governance/src/views/proposal/components/buttons/signOffButton.tsx#L40](https://github.com/solana-labs/oyster/blob/226ab647144f4a9c7739104b82879dc8c136c7b3/packages/governance/src/views/proposal/components/buttons/signOffButton.tsx#L40)
  * rust instruction docs: [https://docs.rs/spl-governance/1.1.1/spl_governance/instruction/fn.sign_off_proposal.html](https://docs.rs/spl-governance/1.1.1/spl_governance/instruction/fn.sign_off_proposal.html)
  
## Vote

* You can now vote on the proposal. Click the blue "Yeah" button on the top right. Since you own all the voting tokens, the proposal passes immediately.
* Relevant resources
  * ts code that casts vote on chain: [https://github.com/solana-labs/oyster/blob/226ab647144f4a9c7739104b82879dc8c136c7b3/packages/governance/src/views/proposal/components/buttons/castVoteButton.tsx#L84](https://github.com/solana-labs/oyster/blob/226ab647144f4a9c7739104b82879dc8c136c7b3/packages/governance/src/views/proposal/components/buttons/castVoteButton.tsx#L84)
  * rust instruction docs: [https://docs.rs/spl-governance/1.1.1/spl_governance/instruction/fn.cast_vote.html](https://docs.rs/spl-governance/1.1.1/spl_governance/instruction/fn.cast_vote.html)
  
## Execute instructions

Once the proposal passes and the hold period has passed (in our case the hold period is zero), you can execute the proposal. To do that you must execute the instructions manually one by one. Anyone can execute the instructions, but only so long as the proposal has passed.

* Press the green play button in the UI
* Run `spl-token balance TOKEN_MINT_ADDESS_HERE`. You should see that we've minted `100` new tokens.
* Relevant resources
  * ts code: [https://github.com/solana-labs/oyster/blob/226ab647144f4a9c7739104b82879dc8c136c7b3/packages/governance/src/views/proposal/components/instruction/buttons/executeInstructionButton.tsx#L65](https://github.com/solana-labs/oyster/blob/226ab647144f4a9c7739104b82879dc8c136c7b3/packages/governance/src/views/proposal/components/instruction/buttons/executeInstructionButton.tsx#L65)
  * rust instruction docs: [https://docs.rs/spl-governance/1.1.1/spl_governance/instruction/fn.execute_instruction.html][https://docs.rs/spl-governance/1.1.1/spl_governance/instruction/fn.execute_instruction.html]


