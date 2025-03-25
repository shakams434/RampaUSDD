
🚀 Rampa USDD - Documentación del Proyecto

📂 Índice
🔹 Pre-requisitos

🔹 Variables de Entorno

🔹 Backend

🔹 Frontend

🔹 Despliegue y Uso

✅ Pre-requisitos

   ✅ Node.js (>= 16.x) → Descárgalo de https://nodejs.org/
   
   ✅ NPM (>= 8.x) → Viene con Node.js
   
   ✅ MetaMask con fondos en la red Amoy
   
   ✅Visual Studio Code
   
   ✅ Git y Git Bash
   
   ✅MetaMask

🔐 Variables de Entorno
Crea un archivo .env en la raíz:

  PRIVATE_KEY=TU_CLAVE_PRIVADA
  INFURA_URL=https://rpc-amoy.polygon.technology
 
🔸 Backend

📁 Carpeta: /backend (o raíz, según tu estructura)

Inicializar el Proyecto Node.js
Ejecuta:

      npm init -y

Esto creará un package.json con la configuración del proyecto.

➡️ 1. Instalar dependencias:

  npm install express cors dotenv ethers

  📌 Explicación de las dependencias:

express → Framework web para crear la API.
cors → Para permitir peticiones desde el frontend.
dotenv → Para manejar variables de entorno.
ethers → Para interactuar con la blockchain.

  
➡️ 2. Crear y configurar el archivo server.js:

  touch server.js
  
📥 Copia y pega el código backend completo.

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

➡️ 3. Iniciar el servidor:

  node server.js
  
El backend quedará corriendo en: http://localhost:3001


🔸 Frontend
📁 Carpeta: /src

➡️ 1. Instalar dependencias:

  npm install react react-dom axios ethers react-scripts
  
➡️ 2. Crear o editar App.js:

  cd src
  touch App.js
  
📥 Pega el código frontend completo.

import React, { useState, useEffect } from "react";
import axios from "axios";
import { ethers } from "ethers";

const API_URL = "http://localhost:3001"; // Tu backend local
const USDD_CONTRACT_ADDRESS = "0xF400b46B9302af87b8e3F831D30967aFe122B297";
const VENTA_CONTRACT_ADDRESS = "0x12736BCB6C7f4c447Ac5aCda50F633FB5F1eEE00";

const ERC20_ABI = [
    "function balanceOf(address owner) view returns (uint256)",
    "function decimals() view returns (uint8)",
    "function transfer(address to, uint256 amount) public returns (bool)"
];

const VENTA_ABI = [
    "function comprarUSDD(address comprador, uint256 montoUSDD) external"
];

const App = () => {
    const [wallet, setWallet] = useState(null);
    const [balanceUSD, setBalanceUSD] = useState(0);
    const [balanceUSDD, setBalanceUSDD] = useState(0);
    const [monto, setMonto] = useState("");
    const [mensaje, setMensaje] = useState("");
    const [historial, setHistorial] = useState([]);

    useEffect(() => {
        detectarWallet();
        if (window.ethereum) {
            window.ethereum.on("accountsChanged", async (accounts) => {
                if (accounts.length > 0) {
                    setWallet(accounts[0]);
                    actualizarBalances(accounts[0]);
                } else {
                    setWallet(null);
                }
            });
        }
    }, []);

    const detectarWallet = async () => {
        if (window.ethereum) {
            try {
                const accounts = await window.ethereum.request({ method: "eth_requestAccounts" });
                setWallet(accounts[0]);
                actualizarBalances(accounts[0]);
            } catch (error) {
                console.error("❌ Error al conectar MetaMask:", error);
            }
        } else {
            alert("⚠️ MetaMask no está instalado.");
        }
    };

    const actualizarBalances = async (direccion) => {
        await consultarBalanceUSD(direccion);
        await consultarBalanceUSDD(direccion);
        await consultarHistorial(direccion);
    };

    const consultarBalanceUSD = async (direccion) => {
        try {
            const { data } = await axios.get(`${API_URL}/balanceUSD`, { params: { usuario: direccion } });
            setBalanceUSD(data.balance);
        } catch (error) {
            console.error("❌ Error consultando balance de USD:", error);
        }
    };

    const consultarBalanceUSDD = async (direccion) => {
        try {
            const provider = new ethers.providers.Web3Provider(window.ethereum);
            const usddToken = new ethers.Contract(USDD_CONTRACT_ADDRESS, ERC20_ABI, provider);
            const decimals = await usddToken.decimals();
            const balance = await usddToken.balanceOf(direccion);
            const formattedBalance = ethers.utils.formatUnits(balance, decimals);
            setBalanceUSDD(parseFloat(formattedBalance));
        } catch (error) {
            console.error("❌ Error consultando balance de USDD:", error);
        }
    };

    const consultarHistorial = async (direccion) => {
        try {
            console.log("📡 Consultando historial de compras en la blockchain...");
            const { data } = await axios.get(`${API_URL}/historialBlockchain`, { params: { usuario: direccion } });

            if (data.length === 0) {
                console.log("⚠️ No se encontraron transacciones para esta wallet.");
            }

            setHistorial(data);
        } catch (error) {
            console.error("❌ Error consultando historial desde la blockchain:", error);
            setHistorial([]);
        }
    };

    const recargarUSD = async () => {
        if (!wallet) return alert("⚠️ Conecta MetaMask primero.");
        try {
            const { data } = await axios.post(`${API_URL}/recargar`, { usuario: wallet, monto: parseFloat(monto) });
            setBalanceUSD(data.balance);
            setMensaje(data.mensaje);
        } catch (error) {
            console.error("❌ Error en la recarga:", error);
        }
    };

    const intercambiarToken = async () => {
        if (!wallet) return alert("⚠️ Conecta MetaMask primero");
        
        try {
            if (balanceUSD < monto) {
                setMensaje("❌ Saldo insuficiente en USD. Recarga primero.");
                return;
            }
    
            const provider = new ethers.providers.Web3Provider(window.ethereum);
            const signer = provider.getSigner();
            const ventaUSDD = new ethers.Contract(VENTA_CONTRACT_ADDRESS, VENTA_ABI, signer);
    
            const { data: precioData } = await axios.get(`${API_URL}/precio`);
            const precioUSDD = parseFloat(precioData.precioUSDD);
            const montoUSDD = monto / precioUSDD;
            const amountToSend = ethers.utils.parseUnits(montoUSDD.toString(), 18);
    
            console.log(`⚡ Aprobando compra de ${montoUSDD} USDD para el contrato de venta...`);
    
            const gasPrice = await provider.getGasPrice();
            const gasLimit = ethers.utils.hexlify(600000);

            const tx = await ventaUSDD.comprarUSDD(wallet, amountToSend, {
                gasLimit: gasLimit,
                maxFeePerGas: gasPrice.mul(2),  
                maxPriorityFeePerGas: gasPrice
            });
            console.log(`⏳ Transacción enviada. Hash: ${tx.hash}`);
    
            await tx.wait();
            console.log("✅ Transacción confirmada!");
    
            setMensaje(`✅ Has recibido ${montoUSDD} USDD en tu wallet. Ver transacción: https://www.oklink.com/amoy/tx/${tx.hash}`);
    
            const { data } = await axios.post(`${API_URL}/registrarTransaccion`, { 
                usuario: wallet, 
                montoUSD: parseFloat(monto), 
                txHash: tx.hash 
            });

            if (data.nuevoBalanceUSD !== undefined) {
                setBalanceUSD(data.nuevoBalanceUSD);  
            }

            await actualizarBalances(wallet);
            setTimeout(() => consultarHistorial(wallet), 3000);  
    
        } catch (error) {
            console.error("❌ Error en la transacción:", error);
            setMensaje("❌ Error en la transacción: " + (error.data?.message || error.message));
        }
    };

    return (
        <div style={{ textAlign: "center", marginTop: "50px" }}>
            <h1>Pasarela de Pagos en Amoy</h1>
            <p><strong>Wallet conectada:</strong> {wallet || "No detectada"}</p>
            <p><strong>Balance en USD (ficticio):</strong> ${balanceUSD}</p>
            <p><strong>Balance en USDD:</strong> {balanceUSDD} USDD</p>
            <input type="number" value={monto} onChange={(e) => setMonto(e.target.value)} placeholder="Monto en USD"/>
            <br />
            <button onClick={recargarUSD}>Recargar USD</button>
            <button onClick={intercambiarToken}>Intercambiar por USDD</button>
            <p>{mensaje}</p>
            <h3>Historial de Transacciones</h3>
            <ul>{historial.map((tx, index) => (<li key={index}><strong>Monto USD:</strong> {tx.montoUSD} → <strong> Hash:</strong> <a href={`https://www.oklink.com/amoy/tx/${tx.txHash}`} target="_blank" rel="noopener noreferrer">{tx.txHash.slice(0, 10)}...</a></li>))}</ul>
        </div>
    );
};

export default App;



➡️ 3. Levantar el frontend:

  npm start

  
Accede desde: http://localhost:5173 (o el puerto que aparezca)

📦 Despliegue y Uso
✅ Abre MetaMask y selecciona la red Polygon Amoy Testnet.

✅ Recarga USD ficticio desde el frontend.

✅ Intercambia por USDD y verifica tu historial.

✅ Consulta saldos y transacciones desde la app.📚 Estructura recomendada para tu README:
