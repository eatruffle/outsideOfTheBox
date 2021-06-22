# OutsideOfTheBox
---

Bellow will be present the most interesting Solidity programming patterns obtained from the network.

## [[Uniswap] Token Staticcall](https://github.com/Uniswap/uniswap-v3-core/blob/main/contracts/UniswapV3Pool.sol)

```solidity
function balance0() private view returns (uint256) {
  (bool success, bytes memory data) =
     token0.staticcall(abi.encodeWithSelector(IERC20Minimal.balanceOf.selector, address(this)));
  require(success && data.length >= 32);
  return abi.decode(data, (uint256));
}
```

## [[Uniswap] Create new contract by assembly](https://github.com/Uniswap/uniswap-v2-core/blob/master/contracts/UniswapV2Factory.sol	)

```solidity
function createPair(address tokenA, address tokenB) external returns (address pair) {
  require(tokenA != tokenB, 'UniswapV2: IDENTICAL_ADDRESSES');
  (address token0, address token1) = tokenA < tokenB ? (tokenA, tokenB) : (tokenB, tokenA);
  require(token0 != address(0), 'UniswapV2: ZERO_ADDRESS');
  require(getPair[token0][token1] == address(0), 'UniswapV2: PAIR_EXISTS'); // single check is sufficient
  bytes memory bytecode = type(UniswapV2Pair).creationCode;
  bytes32 salt = keccak256(abi.encodePacked(token0, token1));
  assembly {
    pair := create2(0, add(bytecode, 32), mload(bytecode), salt)
  }
  IUniswapV2Pair(pair).initialize(token0, token1);
  getPair[token0][token1] = pair;
  getPair[token1][token0] = pair; // populate mapping in the reverse direction
  allPairs.push(pair);
  emit PairCreated(token0, token1, pair, allPairs.length);
}
```
## [[Solidity] Create new contract in solidity](https://docs.soliditylang.org/en/develop/control-structures.html?highlight=require#creating-contracts-via-new)
```solidity
function createDSalted(bytes32 salt, uint arg) public {
  // This complicated expression just tells you how the address
  // can be pre-computed. It is just there for illustration.
  // You actually only need ``new D{salt: salt}(arg)``.
  address predictedAddress = address(uint160(uint(keccak256(abi.encodePacked(
            bytes1(0xff),
            address(this),
            salt,
            keccak256(abi.encodePacked(
                type(D).creationCode,
                arg
            ))
  )))));

  D d = new D{salt: salt}(arg);
  require(address(d) == predictedAddress);
    }
}
```
## [[AAVE] Bitwise operation for fix specific initial conditions](protocol-v2-ice-mainnet-deployment-03-12-2020\contracts\protocol\libraries\configuration\ReserveConfiguration)

```solidity
    //bit 0-15: LTV
    //bit 16-31: Liq. threshold
    //bit 32-47: Liq. bonus
    //bit 48-55: Decimals
    //bit 56: Reserve is active
    //bit 57: reserve is frozen
    //bit 58: borrowing is enabled
    //bit 59: stable rate borrowing enabled
    //bit 60-63: reserved
    //bit 64-79: reserve factor
	
  uint256 constant LTV_MASK =                   0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF0000; // prettier-ignore
  uint256 constant LIQUIDATION_THRESHOLD_MASK = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF0000FFFF; // prettier-ignore
  uint256 constant LIQUIDATION_BONUS_MASK =     0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF0000FFFFFFFF; // prettier-ignore
  uint256 constant DECIMALS_MASK =              0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF00FFFFFFFFFFFF; // prettier-ignore
  uint256 constant ACTIVE_MASK =                0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEFFFFFFFFFFFFFF; // prettier-ignore
  uint256 constant FROZEN_MASK =                0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFDFFFFFFFFFFFFFF; // prettier-ignore
  uint256 constant BORROWING_MASK =             0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFBFFFFFFFFFFFFFF; // prettier-ignore
  uint256 constant STABLE_BORROWING_MASK =      0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF7FFFFFFFFFFFFFF; // prettier-ignore
  uint256 constant RESERVE_FACTOR_MASK =        0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF0000FFFFFFFFFFFFFFFF; // prettier-ignore

	
  function getParams(DataTypes.ReserveConfigurationMap storage self)
    internal
    view
    returns (
      uint256,
      uint256,
      uint256,
      uint256,
      uint256
    )
  {
    uint256 dataLocal = self.data;

    return (
      dataLocal & ~LTV_MASK,
      (dataLocal & ~LIQUIDATION_THRESHOLD_MASK) >> LIQUIDATION_THRESHOLD_START_BIT_POSITION,
      (dataLocal & ~LIQUIDATION_BONUS_MASK) >> LIQUIDATION_BONUS_START_BIT_POSITION,
      (dataLocal & ~DECIMALS_MASK) >> RESERVE_DECIMALS_START_BIT_POSITION,
      (dataLocal & ~RESERVE_FACTOR_MASK) >> RESERVE_FACTOR_START_BIT_POSITION
    );
  }
  
      (, , vars.liquidationBonus, vars.collateralDecimals, ) = collateralReserve
      .configuration
      .getParams();

function getFlags(DataTypes.ReserveConfigurationMap storage self)
    internal
    view
    returns (
      bool,
      bool,
      bool,
      bool
    )
  {
    uint256 dataLocal = self.data;

    return (
      (dataLocal & ~ACTIVE_MASK) != 0,
      (dataLocal & ~FROZEN_MASK) != 0,
      (dataLocal & ~BORROWING_MASK) != 0,
      (dataLocal & ~STABLE_BORROWING_MASK) != 0
    );
  }
  
(bool isActive, bool isFrozen, , ) = reserve.configuration.getFlags();
```

## [[OpenZeppelin] Slightly reduce ERC721, ERC20 deployment gas cost](https://github.com/OpenZeppelin/openzeppelin-contracts/issues/2665)

### ERC721
```solidity
  require(to != owner, "ERC721: approval to current owner");
  
  //to
  
   require(owner != to, "ERC721: approval to current owner");
```
### ERC20
```solidity
_approve(_msgSender(), spender, _allowances[_msgSender()][spender] + addedValue);

//to

_approve(_msgSender(), spender, addedValue + _allowances[_msgSender()][spender]);
```
## [[OpenZeppelin] ERC721 tree](https://github.com/OpenZeppelin/openzeppelin-contracts/tree/master/contracts/token/ERC721)

![ERC721-080](https://user-images.githubusercontent.com/85684666/122900814-4811ca00-d34d-11eb-85f6-0e810a5290ab.png)
