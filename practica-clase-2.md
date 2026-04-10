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

## Direcciones de referencia — Base Sepolia (Aave V3)

La testnet actual de Base es **Base Sepolia** (chain id **84532**). Las direcciones siguientes provienen del [aave-address-book de BGD Labs](https://github.com/bgd-labs/aave-address-book/blob/main/src/AaveV3BaseSepolia.sol) (convención: verificar en el repo si hubiera un redeploy).

| Qué es | Dirección |
|--------|-----------|
| `PoolAddressesProvider` (argumento del constructor) | `0xE4C23309117Aa30342BFaae6c95c6478e0A4Ad00` |
| `Pool` (comprobación: `pool()` en tu wrapper o `getPool()` en el provider) | `0x8bAB6d1b75f19e9eD9fCe8b9BD338844fF79aE27` |
| WETH subyacente | `0x4200000000000000000000000000000000000006` |
| USDC subyacente (6 decimales) | `0xba50Cd2A20f6DA35D788639E581bca8d0B5d4D5f` |

**ETH nativo → WETH:** el `deposit` del práctico usa el **contrato ERC-20 WETH**. Si solo tienes ETH, en el contrato WETH llama a **`deposit()`** enviando **value** en wei en la misma transacción para recibir WETH.

**Fuentes de verdad para otras redes:** archivos `AaveV3<Red>.sol` en el mismo repositorio (`POOL_ADDRESSES_PROVIDER`).

---

## Flujo de práctica sugerido (paso a paso)

1. **Elegir market y provider**
   - Selecciona red + market (por ejemplo, *Aave V3 Ethereum* o *Aave V3 Base Sepolia*).
   - Usa el `PoolAddressesProvider` de **esa misma red** (tabla anterior para Base Sepolia; [address book](https://github.com/bgd-labs/aave-address-book/tree/main/src) para el resto).
2. **Deploy**
   - Despliega `AaveBasicInteractor(provider)`.
   - Anota la dirección del contrato desplegado: es tu **wrapper**.
3. **Supply (`deposit`)**
   - El `approve` **no** se hace “al wrapper como contrato que expone approve”: se llama **`approve` en el contrato del token** (WETH, USDC, …).
   - Parámetros: `spender` = dirección del **wrapper**; `amount` ≥ cantidad que pasarás a `deposit` (mismas unidades mínimas del token).
   - Luego `deposit(asset, amount)` desde la **misma cuenta** que firmó el `approve`.
4. **Borrow**
   - Llama a `borrow(assetToBorrow, amount, rateMode)`.
   - En muchos mercados, modo variable = **`2`** (`interestRateMode`).
   - **GHO**: solo en mercados donde exista la reserva GHO y tu colateral lo permita (en Base Sepolia puede no aplicar igual que en mainnet).
5. **Repay**
   - Otra vez: `approve` del **token adeudado** al **wrapper**, luego `repay(assetBorrowed, amount, rateMode)`.
6. **Withdraw**
   - `withdraw(assetSupplied, amount)` si el HF y la liquidez lo permiten.

---

## Pruebas con Remix IDE (Base Sepolia)

1. **Red y wallet:** MetaMask (u otra wallet) en **Base Sepolia**; ETH de faucet para gas.
2. **Remix:** [remix.ethereum.org](https://remix.ethereum.org) → **Deploy & run** → **Injected Provider**. Comprueba que Remix muestra la red correcta.
3. **Compilar** `AaveBasicInteractor.sol` (compilador acorde a `^0.8.20`).
4. **Deploy** del contrato con el constructor: `0xE4C23309117Aa30342BFaae6c95c6478e0A4Ad00` (Base Sepolia).
5. **`approve` al wrapper desde Remix**
   - Compila una interfaz mínima con `function approve(address,uint256) external returns (bool);`.
   - **At Address** con la dirección del **token** (no del wrapper).
   - Llama `approve(spender, amount)` con `spender` = dirección del **wrapper** desplegado.
6. **Argumentos `uint256`:** Remix/ethers no aceptan decimales. `amount` = entero en unidad mínima del token:  
   `cantidad_en_tokens × 10^decimales` (WETH 18, USDC en esta red 6).

La **Remix VM** no tiene Aave desplegado; para tocar el Pool real necesitas **testnet/mainnet** con Injected Provider.

---

## Errores típicos

- Usar una dirección de `Pool` hardcodeada en vez de resolverla desde `PoolAddressesProvider`.
- Olvidar `approve` en el **token** antes de `deposit` o `repay`, o poner como `spender` una dirección que no es el **wrapper actual**.
- **Volver a desplegar** el wrapper y seguir usando el `approve` viejo (cada deploy tiene dirección nueva).
- Pasar **`0.00001`** u otra cadena con punto en Remix para un `uint256` → usar solo el **entero** en unidades mínimas.
- Intentar `withdraw` cuando todavía sostienes deuda (o el health factor quedaría por debajo de 1).

### Depuración en el explorador (p. ej. Basescan)

- **`status: 0` / execution failed** con **poco gas usado** (~decenas de miles): suele ser revierte **temprana**, muy a menudo `transferFrom failed` (saldo o `allowance` insuficiente, o `spender` incorrecto).
- **`logs: []`** en una tx fallida es coherente con revert antes de emitir eventos del Pool.
- Comprueba en el contrato del token: `balanceOf(tu_cuenta)` y `allowance(tu_cuenta, wrapper)`.

### Error de wallet / RPC (`BAD_DATA`, `result: null`, hash inválido)

Suele ser **respuesta vacía del RPC** o glitch de la extensión, no el contrato. Prueba: otro **RPC URL** de Base Sepolia en la wallet, recargar Remix, desconectar y volver a conectar la wallet, revisar si la transacción aparece igual en la actividad de MetaMask.

