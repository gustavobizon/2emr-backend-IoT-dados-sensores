-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
> Primeiro são importadas as bibliotecas utilizando:

	const express = require('express');
	const sqlite3 = require('sqlite3').verbose();
	const bcrypt = require('bcrypt');
	const jwt = require('jsonwebtoken');
	const cors = require('cors');
	const http = require('http'); // Importa o módulo http
	const { Server } = require('socket.io'); // Importa o Socket.IO
	const app = express();
	const PORT = 3000;
	const SECRET_KEY = 'sua_chave_secreta';

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

> O código para fazer a conexão entre o backend e o frontend pelo http:

	const server = http.createServer(app);
	const io = new Server(server, {
	    cors: {
	        origin: '*', // Habilita CORS para qualquer origem
	        methods: ['GET', 'POST'],
	        }

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

> Rota para a criação do banco de dados:

	// Banco de dados e rotas (o restante do código permanece o mesmo)
	const db = new sqlite3.Database('banco-de-dados.db');
	// Criação das tabelas (o código permanece o mesmo)
	db.serialize(() => {
	    db.run(`CREATE TABLE IF NOT EXISTS usuarios (
	        id INTEGER PRIMARY KEY AUTOINCREMENT,
	        username TEXT UNIQUE,
	        password TEXT
	    )`);
	    db.run(`CREATE TABLE IF NOT EXISTS dados_sensores (
	        id INTEGER PRIMARY KEY AUTOINCREMENT,
	        sensor_id INTEGER,
	        temperatura REAL,
	        umidade REAL,
	        timestamp DATETIME DEFAULT CURRENT_TIMESTAMP
	    )`);
	});

----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

> Rota para buscar dados dos sensores:

	// Rota para buscar todos os dados dos sensores (protegida por JWT)
	app.get('/dados-sensores', authenticateJWT, (req, res) => {
	    const query = `SELECT * FROM dados_sensores`;
	    db.all(query, [], (err, rows) => {
	        if (err) {
	            console.error('Erro ao buscar dados no banco de dados:', err.message);
	            res.status(500).send('Erro ao buscar os dados.');
	        } else {
	            res.json(rows);
	        }
	    });
	});

> Rota para buscar dados dos sensores com tempo:

	app.get('/dados-sensores/tempo', authenticateJWT, (req, res) => {
	    const { inicio, fim } = req.query; // Espera que os parâmetros sejam passados na URL
	    if (!inicio || !fim) {
	        return res.status(400).json({ message: 'Os parâmetros de data "inicio" e "fim" são obrigatórios.' });
	    }
	    const query = `SELECT * FROM dados_sensores WHERE timestamp BETWEEN ? AND ?`;
	    db.all(query, [inicio, fim], (err, rows) => {
	        if (err) {
	            console.error('Erro ao buscar dados no banco de dados:', err.message);
	            res.status(500).send('Erro ao buscar os dados.');
	        } else {
	            res.json(rows);
	        }
	    });
	});

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

> Rota para inserir dados nos sensores:

	app.post('/dados-sensores', async (req, res) => {
	    const dados = req.body;
	    console.log('Dados recebidos dos sensores:', dados);
	    try {
	        await addSensorData(dados); // Chama a função para inserir dados e emitir evento
	        res.send('Dados recebidos e armazenados com sucesso.');
	    } catch (err) {
	        console.error('Erro ao inserir dados no banco de dados:', err.message);
	        res.status(500).send('Erro ao processar os dados.');
	    }
	});

 > Função para inserir dados e ativar evento:

	async function addSensorData(newData) {
	    // Insira no banco de dados
	    return new Promise((resolve, reject) => {
	        db.run(`INSERT INTO dados_sensores (sensor_id, temperatura, umidade) VALUES (?, ?, ?)`,
	            [newData.sensor_id, newData.temperatura, newData.umidade],
	            (err) => {
	                if (err) {
	                    return reject(err); // Se houver erro, rejeita a promessa
	                }
	                console.log('Dados inseridos no banco de dados com sucesso.');
	                io.emit('sensorDataUpdate', newData); // Emitindo os dados atualizados
	                resolve(); // Se tudo correr bem, resolve a promessa
	            });
	    });
	}

----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

> Rota para limpar os dados na tabela:

	// Rota para limpar todos os dados da tabela (protegida por JWT)
	app.delete('/limpar-dados', authenticateJWT, (req, res) => {
	    const query = `DELETE FROM dados_sensores`;
	    db.run(query, [], (err) => {
	        if (err) {
	            console.error('Erro ao limpar dados do banco de dados:', err.message);
	            res.status(500).send('Erro ao limpar os dados.');
	        } else {
	            console.log('Dados da tabela limpos com sucesso.');
	            res.send('Dados da tabela foram limpos com sucesso.');
	        }
	    });
	});

 ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

>
 	server.listen(PORT, () => {
	    console.log(`Servidor rodando em http://localhost:${PORT}`);
	});
