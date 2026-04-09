# Práctica (Clase 2) — Aave V3: approve → supply → borrow → repay → withdraw

Este documento contiene el **código mínimo** (educativo, no productivo) para practicar interacciones básicas con Aave V3 desde Solidity.

## Qué vas a implementar

- **Resolver el `Pool` correctamente** desde `PoolAddressesProvider` (evita direcciones obsoletas).
- **Interacciones mínimas** con `IPool`: `supply`, `borrow`, `repay`, `withdraw`.
- **Patrón ERC-20**: `transferFrom` + `approve` antes de `supply` y `repay`.
- **Caso GHO**: para el integrador, es un `borrow` de la reserva `GHO` en el market correspondiente (si el colateral/mercado lo permite).

## Foundry (opcional, recomendado)

Si estás usando Foundry en local:

```bash
forge init aave-clase-2
cd aave-clase-2
mkdir -p src
```

Luego copia el archivo `AaveBasicInteractor.sol` de abajo a `src/AaveBasicInteractor.sol`.

## Código (Solidity)

Guarda esto como `src/AaveBasicInteractor.sol`.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/// @notice ERC-20 mínimo para la práctica.
interface IERC20 {
    function approve(address spender, uint256 amount) external returns (bool);
    function transferFrom(address from, address to, uint256 amount) external returns (bool);
    function balanceOf(address account) external view returns (uint256);
}

/// @notice Interfaz mínima del Pool de Aave V3 para la práctica.
interface IPool {
    function supply(
        address asset,
        uint256 amount,
        address onBehalfOf,
        uint16 referralCode
    ) external;

    function borrow(
        address asset,
        uint256 amount,
        uint256 interestRateMode,
        uint16 referralCode,
        address onBehalfOf
    ) external;

    function repay(
        address asset,
        uint256 amount,
        uint256 interestRateMode,
        address onBehalfOf
    ) external returns (uint256);

    function withdraw(
        address asset,
        uint256 amount,
        address to
    ) external returns (uint256);
}

/// @notice Registry del mercado: te permite descubrir la dirección vigente del Pool.
interface IPoolAddressesProvider {
    function getPool() external view returns (address);
}

/// @notice Wrapper educativo. NO es código productivo.
/// - No maneja permisos avanzados.
/// - No optimiza gas.
/// - No hace validaciones de mercado (caps, paused, etc.).
/// Objetivo: practicar el flujo mental y el patrón approve + llamadas a Pool.
contract AaveBasicInteractor {
    IPool public immutable pool;

    /// @param provider Dirección del PoolAddressesProvider del market elegido (por red).
    constructor(address provider) {
        address poolAddr = IPoolAddressesProvider(provider).getPool();
        pool = IPool(poolAddr);
    }

    /// @notice deposit = transferFrom (usuario → wrapper) + approve (wrapper → pool) + supply(onBehalfOf = usuario)
    function deposit(address asset, uint256 amount) external {
        require(amount > 0, "amount=0");

        // 1) Trae tokens al wrapper
        require(IERC20(asset).transferFrom(msg.sender, address(this), amount), "transferFrom failed");

        // 2) Autoriza al Pool a mover tokens desde el wrapper
        require(IERC20(asset).approve(address(pool), amount), "approve failed");

        // 3) Supply en nombre del usuario
        pool.supply(asset, amount, msg.sender, 0);
    }

    /// @notice borrow no requiere approve porque el usuario RECIBE fondos.
    /// @param interestRateMode típicamente 2 (variable) en muchos mercados, pero depende del mercado.
    function borrow(address asset, uint256 amount, uint256 interestRateMode) external {
        require(amount > 0, "amount=0");
        pool.borrow(asset, amount, interestRateMode, 0, msg.sender);
    }

    /// @notice repay = transferFrom (usuario → wrapper) + approve (wrapper → pool) + repay(onBehalfOf = usuario)
    function repay(address asset, uint256 amount, uint256 interestRateMode) external returns (uint256 repaid) {
        require(amount > 0, "amount=0");

        require(IERC20(asset).transferFrom(msg.sender, address(this), amount), "transferFrom failed");
        require(IERC20(asset).approve(address(pool), amount), "approve failed");

        repaid = pool.repay(asset, amount, interestRateMode, msg.sender);
    }

    /// @notice withdraw redime aTokens del usuario. Puede fallar si la posición queda insegura o no hay liquidez.
    function withdraw(address asset, uint256 amount) external returns (uint256 withdrawn) {
        withdrawn = pool.withdraw(asset, amount, msg.sender);
    }

    /// @notice Helper para ver saldos del wrapper (útil para depurar la práctica).
    function wrapperBalance(address asset) external view returns (uint256) {
        return IERC20(asset).balanceOf(address(this));
    }
}
```

## Flujo de práctica sugerido (paso a paso)

1. **Elegir market y provider**
   - Selecciona red + market (por ejemplo, *Aave V3 Ethereum Market*).
   - Obtén la dirección del `PoolAddressesProvider` para esa red/market (docs oficiales).
2. **Deploy**
   - Despliega `AaveBasicInteractor(provider)`.
3. **Supply**
   - En la UI o en un script, haz `approve(token, wrapper, amount)` (al wrapper) si vas a usar `transferFrom` hacia el wrapper.
   - Llama a `deposit(asset, amount)`.
4. **Borrow**
   - Llama a `borrow(assetToBorrow, amount, rateMode)`.
   - **GHO**: si la reserva existe y tu colateral lo permite, `assetToBorrow = GHO`.
5. **Repay**
   - Haz `approve(tokenBorrowed, wrapper, amount)` y luego `repay(assetBorrowed, amount, rateMode)`.
6. **Withdraw**
   - Intenta `withdraw(assetSupplied, amount)` (si el HF y la liquidez lo permiten).

## Errores típicos

- Usar una dirección de `Pool` hardcodeada en vez de resolverla desde `PoolAddressesProvider`.
- Olvidar `approve` antes de `deposit` o `repay`.
- Intentar `withdraw` cuando todavía sostienes deuda (o el HF quedaría < 1).

