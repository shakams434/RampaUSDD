1️⃣ Desplegar los contratos inteligentes

📌 Pasos clave:
✅ Diseñar y desarrollar los contratos en Solidity.
✅ Desplegarlos en una testnet como Polygon Amoy, Sepolia o Goerli.
✅ Guardar las direcciones de los contratos desplegados.


1️⃣ Desplegar contratos en blockchain  ➝  2️⃣ Desarrollar backend con API  ➝  3️⃣ Crear frontend con React


🔹 Paso 1: Configurar el Entorno
Primero, asegúrate de tener instaladas las herramientas necesarias:

✅ Node.js (versión 16 o superior)

✅ Metamask con la red de prueba Polygon Amoy

✅ Git Bash y Visual Studio Code

🔹 Paso 2: Crear el Proyecto en Hardhat
Hardhat es una herramienta que nos permite compilar, probar y desplegar contratos inteligentes.

1️⃣ Abre Git Bash y crea una carpeta para el proyecto

        mkdir venta-usdd
        cd venta-usdd

2️⃣ Inicializa un proyecto en Hardhat

        npx hardhat

Selecciona "Create a JavaScript project", y acepta todas las opciones por defecto.

3️⃣ Instala las dependencias necesarias

        npm install --save-dev hardhat @nomicfoundation/hardhat-toolbox dotenv
        npm install @openzeppelin/contracts
        npm install @chainlink/contracts

Glosario:
hardhat: Entorno de desarrollo para compilar, probar y desplegar smart contracts en Ethereum.

@nomicfoundation/hardhat-toolbox: Conjunto de herramientas útiles para pruebas, depuración y análisis en Hardhat.

dotenv: Permite cargar variables de entorno desde un archivo .env.

@openzeppelin/contracts: Librería con contratos inteligentes seguros y reutilizables (como ERC-20, ERC-721).

@chainlink/contracts: Contratos para interactuar con oráculos de Chainlink (por ejemplo, para obtener precios externos).


4️⃣ Configura el archivo .env para almacenar las claves privadas y el RPC de Polygon Amoy.
📌 Crea un archivo .env en la raíz del proyecto con el siguiente contenido:

        PRIVATE_KEY=TU_CLAVE_PRIVADA
        INFURA_URL=https://rpc-amoy.polygon.technology
        
⚠️ No compartas tu clave privada con nadie.

🔹 Paso 3: Agregar los Contratos
📌 Crea una carpeta contracts dentro de venta-usdd y coloca dentro los siguientes contratos:

📌 Contrato USDD_Token.sol
📍 Ubicación: contracts/USDD_Token.sol

solidity

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

// Importa implementación del token ERC20
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
// Importa control de propiedad (soloOwner)
import "@openzeppelin/contracts/access/Ownable.sol";

// Contrato del token USDD, tipo ERC20, con control de dueño
contract USDD_Token is ERC20, Ownable {
    // Precio estable simulado (1 USDD = 1 USD ficticio)
    uint256 public stablePrice = 1 * 10**18;

    // Constructor: define nombre y símbolo, y hace mint inicial al dueño
    constructor() ERC20("USDD Token", "USDD") Ownable(msg.sender) {
        _mint(msg.sender, 30_000_000 * 10 ** decimals());
    }

    // Función para acuñar tokens a una dirección (solo el dueño)
    function mint(address to, uint256 amount) external onlyOwner {
        _mint(to, amount);
    }

    // Función para quemar tokens del dueño (solo el dueño)
    function burn(uint256 amount) external onlyOwner {
        _burn(msg.sender, amount);
    }

    // Establece un nuevo precio estable (solo el dueño)
    function setStablePrice(uint256 newPrice) external onlyOwner {
        require(newPrice > 0, "El precio debe ser mayor a 0");
        stablePrice = newPrice;
    }

    // Aprueba a otro contrato para gastar una cantidad (solo el dueño)
    function approveSale(address saleContract, uint256 amount) external onlyOwner {
        _approve(msg.sender, saleContract, amount);
    }
}

📌 Contrato VentaUSDD.sol
📍 Ubicación: contracts/VentaUSDD.sol


        // SPDX-License-Identifier: MIT
        pragma solidity ^0.8.20;
        
        import "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";
        import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
        
        contract VentaUSDD {
            AggregatorV3Interface internal priceFeed;
            IERC20 public usddToken;
            address public owner;
        
            event CompraUSDD(address indexed comprador, uint256 montoUSD, uint256 montoUSDD);
        
            constructor(address _usddToken, address _priceFeed) {
                usddToken = IERC20(_usddToken);
                priceFeed = AggregatorV3Interface(_priceFeed);
                owner = msg.sender;
            }
        
            function getLatestPrice() public view returns (uint256) {
                (, int price, , , ) = priceFeed.latestRoundData();
                require(price > 0, "Precio invalido");
                return uint256(price) * 10 ** 10; // Convertimos a 18 decimales
            }
        
            function comprarUSDD(address comprador, uint256 montoUSD) external {
                uint256 precioUSDD = getLatestPrice();
                uint256 montoUSDD = (montoUSD * 1e18) / precioUSDD;
        
                require(usddToken.balanceOf(address(this)) >= montoUSDD, "Contrato sin fondos");
        
                usddToken.transfer(comprador, montoUSDD);
                emit CompraUSDD(comprador, montoUSD, montoUSDD);
            }
        }

🔹 Paso 4: Configurar Hardhat para el Despliegue
📍 Ubicación: hardhat.config.js
✍ Edita el archivo hardhat.config.js y reemplázalo con:


        require("@nomicfoundation/hardhat-toolbox");
        require("dotenv").config();
        
        module.exports = {
          solidity: "0.8.20",
          networks: {
            amoy: {
              url: process.env.INFURA_URL,
              accounts: [process.env.PRIVATE_KEY]
            }
          }
        };

🔹 Paso 5: Crear los Scripts de Despliegue
📌 Crea una carpeta scripts dentro del proyecto y agrega los siguientes archivos:

📌 Desplegar USDD_Token
📍 Ubicación: scripts/deployUSDD.js


// Importa el entorno de Hardhat Runtime Environment (hre)
const hre = require("hardhat");

async function main() {
  const PRUEBA = await hre.ethers.getContractFactory("Prueba_Token");
  const prueba = await PRUEBA.deploy();

  await prueba.waitForDeployment(); // ✅ asegura que ya fue minado
  const deployedAddress = await prueba.getAddress(); // ✅ obtiene la dirección

  console.log(`✅ PRUEBA Token desplegado en: ${deployedAddress}`);
}

main().catch((error) => {
  console.error(error);
  process.exit(1);
});
const hre = require("hardhat");

async function main() {
  const PRUEBA = await hre.ethers.getContractFactory("Prueba_Token");
  const prueba = await PRUEBA.deploy();

  await prueba.waitForDeployment(); // ✅ asegura que ya fue minado
  const deployedAddress = await prueba.getAddress(); // ✅ obtiene la dirección

  console.log(`✅ PRUEBA Token desplegado en: ${deployedAddress}`);
}

main().catch((error) => {
  console.error(error);
  process.exit(1);
});






📌 Desplegar VentaUSDD
📍 Ubicación: scripts/deployVenta.js


        const hre = require("hardhat");
        
        async function main() {
          const usddAddress = "DIRECCIÓN_DEL_TOKEN_USDD";  // Reemplazar con la dirección real del token USDD
          const priceFeed = "DIRECCIÓN_DEL_ORÁCULO_CHAINLINK"; // Reemplazar con dirección real
        
          const VentaUSDD = await hre.ethers.getContractFactory("VentaUSDD");
          const ventaUSDD = await VentaUSDD.deploy(usddAddress, priceFeed);
        
          await ventaUSDD.deployed();
          console.log(`✅ Contrato de VentaUSDD desplegado en: ${ventaUSDD.address}`);
        }
        
        main().catch((error) => {
          console.error(error);
          process.exit(1);
        });
        
🔹 Paso 6: Desplegar los Contratos
1️⃣ Compila los contratos para verificar que todo esté correcto:

        npx hardhat compile

2️⃣ Desplega el Token USDD en Polygon Amoy:

        npx hardhat run scripts/deployUSDD.js --network amoy

📌 Guarda la dirección generada.

3️⃣ Edita scripts/deployVenta.js con la dirección del contrato USDD y el oráculo de Chainlink.

4️⃣ Desplega el contrato de VentaUSDD:

        npx hardhat run scripts/deployVenta.js --network amoy
        
📌 Guarda la dirección del contrato de Venta.

🔹 Paso 7: Verificar Contratos en Polygonscan
💡 (Opcional pero recomendado)
Para verificar el código en Polygonscan:

npx hardhat verify --network amoy DIRECCIÓN_DEL_CONTRATO ARGUMENTOS_SI_APLICAN

🚀 Conclusión
¡Listo! Ya tienes ambos contratos desplegados en la blockchain y listos para integrarse con el backend. 🏗️
📌 Próximo paso: Configurar y desplegar el backend para que interactúe con estos contratos.
