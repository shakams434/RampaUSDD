# RampaUSDD
Rampa fiat a criptoactivo


1️⃣ Antes de comenzar, asegúrate de tener instalados los siguientes programas en tu máquina:

✅ Node.js (>= 16.x) → Descárgalo de https://nodejs.org/

✅ NPM (>= 8.x) → Viene con Node.js

✅ MetaMask con fondos en la red Amoy


 Crear una Carpeta

2️⃣ Inicializar el Proyecto Node.js
Ejecuta:

      npm init -y

Esto creará un package.json con la configuración del proyecto.

3️⃣ Instalar Dependencias
Ahora instala las librerías necesarias:

      npm install express cors dotenv ethers

📌 Explicación de las dependencias:

express → Framework web para crear la API.
cors → Para permitir peticiones desde el frontend.
dotenv → Para manejar variables de entorno.
ethers → Para interactuar con la blockchain.

4️⃣ Configurar Variables de Entorno
Crea un archivo llamado .env en la raíz del proyecto:

      touch .env

Ábrelo y agrega:

      PRIVATE_KEY=TU_CLAVE_PRIVADA
      INFURA_URL=https://rpc-amoy.polygon.technology

5️⃣ Crear el Archivo del Servidor
Crea un archivo llamado server.js:

       touch server.js

Abre server.js y copia y pega el siguiente código:


        require("dotenv").config();
        const express = require("express");
        const cors = require("cors");
        const { ethers } = require("ethers");
        
        const app = express();
        app.use(cors());
        app.use(express.json());
        
        const provider = new ethers.providers.JsonRpcProvider(process.env.INFURA_URL);
        const wallet = new ethers.Wallet(process.env.PRIVATE_KEY, provider);
        const signer = wallet.connect(provider);
        
        // 📌 Direcciones de los contratos
        const USDD_CONTRACT_ADDRESS = "0xF400b46B9302af87b8e3F831D30967aFe122B297";
        const VENTA_CONTRACT_ADDRESS = "0x12736BCB6C7f4c447Ac5aCda50F633FB5F1eEE00";
        const CHAINLINK_FEED_ADDRESS = "0x1b8739bB4CdF0089d07097A9Ae5Bd274b29C6F16";
        
        // 📌 ABIs
        const ERC20_ABI = [
            "function transfer(address to, uint256 amount) public returns (bool)",
            "function balanceOf(address owner) view returns (uint256)",
            "function decimals() view returns (uint8)"
        ];
        
        const VENTA_ABI = [
            "function comprarUSDD(address comprador, uint256 montoUSD) external",
            "event CompraUSDD(address indexed comprador, uint256 montoUSD, uint256 montoUSDD)"
        ];
        
        const CHAINLINK_FEED_ABI = [
            "function latestRoundData() public view returns (uint80, int256, uint256, uint256, uint80)"
        ];
        
        // 📌 Instancias de los contratos
        const usddToken = new ethers.Contract(USDD_CONTRACT_ADDRESS, ERC20_ABI, signer);
        const ventaUSDD = new ethers.Contract(VENTA_CONTRACT_ADDRESS, VENTA_ABI, signer);
        const priceFeed = new ethers.Contract(CHAINLINK_FEED_ADDRESS, CHAINLINK_FEED_ABI, provider);
        
        let transacciones = {};
        let balances = {};
        
        // 📌 Recargar USD ficticio
        app.post("/recargar", (req, res) => {
            const { usuario, monto } = req.body;
            if (!usuario || !monto) return res.status(400).json({ error: "Datos inválidos" });
        
            if (!balances[usuario]) balances[usuario] = 0;
            balances[usuario] += parseFloat(monto);
        
            res.json({ mensaje: `✅ Recargaste $${monto}. Nuevo balance: $${balances[usuario]}`, balance: balances[usuario] });
        });
        
        // 📌 Obtener balance en USD ficticio
        app.get("/balanceUSD", (req, res) => {
            const { usuario } = req.query;
            if (!usuario) return res.status(400).json({ error: "Falta el usuario" });
        
            const balance = balances[usuario] || 0;
            res.json({ balance });
        });
        
        // 📌 Obtener precio en tiempo real del oráculo Chainlink
        app.get("/precio", async (req, res) => {
            try {
                const [, answer] = await priceFeed.latestRoundData();
                const price = ethers.utils.formatUnits(answer, 8);
                res.json({ precioUSDD: price });
            } catch (error) {
                res.status(500).json({ error: error.message });
            }
        });
        
        // 📌 Registrar transacción
        app.post("/registrarTransaccion", async (req, res) => {
            const { usuario, montoUSD, txHash } = req.body;
        
            try {
                if (!balances[usuario] || balances[usuario] < montoUSD) {
                    return res.status(400).json({ error: "❌ Saldo insuficiente en USD. Recarga primero." });
                }
        
                balances[usuario] -= montoUSD;
        
                if (!transacciones[usuario]) transacciones[usuario] = [];
                transacciones[usuario].push({ usuario, montoUSD, txHash, fecha: new Date().toISOString() });
        
                res.json({ mensaje: "✅ Transacción registrada en el backend.", nuevoBalanceUSD: balances[usuario] });
        
            } catch (error) {
                res.status(500).json({ error: error.message });
            }
        });
        
        const PORT = process.env.PORT || 3001;
        app.listen(PORT, () => console.log(`🚀 Servidor backend corriendo en puerto ${PORT}`));


6️⃣ Ejecutar el Servidor
Una vez que tengas el código, inicia el backend con:

      node server.js

Deberías ver un mensaje como:

      🚀 Servidor backend corriendo en puerto 3001

7️⃣ Probar API con Postman o cURL
Puedes probar las rutas del backend usando Postman o cURL.

Ejemplo para recargar saldo:

      curl -X POST http://localhost:3001/recargar -H "Content-Type: application/json" -d '{"usuario": "0xTuDireccion", "monto": 10


🎉 ¡Listo!
