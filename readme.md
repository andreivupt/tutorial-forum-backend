## Construção do backend para uma aplicação de fórum

#### Criando o banco de dados
* Abrir programa Workbench
* Copie os comandos a seguir e execute no Workbench
* Pressione as teclas 'Ctrl + Enter' para executatar os comandos
<hr>

* Criar database
```
create database app_forum_db;
```
* Definir a database
```
use app_forum_db;
```
* Criar tabela users
```
create table users (
	id int not null auto_increment,
    name varchar(120) not null,
    email varchar(120) not null,
    password varchar(120) not null,
    status enum('A','I') default ('A'),
    created_at datetime default current_timestamp,
    primary key(id)
);
```
* Criar tabela posts
```
create table posts (
    id int not null auto_increment,
    descricao varchar(255) not null,
    user_id int not null,
    created_at datetime default current_timestamp,
    primary key(id)
);
```
* Criar tabela comments
```
create table comments(
    id int not null auto_increment,
    descricao varchar(120) not null,
    post_id int not null,
    user_id int not null,
    primary key(id),
    foreign key (post_id) references posts(id),
    foreign key (user_id) references users(id)
);
```
* Criar tabela reactions
```
create table reactions(
	id int not null auto_increment,
    user_id int not null,
    post_id int not null,
    comment_id int,
    primary key(id),
    foreign key (post_id) references posts(id),
    foreign key (user_id) references users(id)
);
```

#### Passo a passo do desenvolvimento utilizando gitBash
* Clonar o repositório no computador
* Acessar a pasta
* Abrir a pasta no VSCode

#### Configurações iniciais
* Executar o comando 'npm i' para importar os pacotes da aplicação
```
npm i 
```

* Criar o arquivo '.env' na raiz do projeto e colar as variáveis do arquivo '.env.example'.
* Adicionar os valores às variáveis

#### Configurar inicialização da API
* Abrir o arquivo 'app.js' e colar o código
```
const express         = require('express');
const cors            = require('cors');
const app             = express();
require('dotenv').config();

app.use(cors());
app.use(express.json());
app.set('port', process.env.PORT);

module.exports = app;
```

* Abrir o arquivo 'server.js' e colar o código
```
const app = require('./app');
const port = app.get('port');

app.listen(port, () => console.log(`Running at port ${ port }`));
```
### Testar servidor
* Executar o comando com o gitBash
```
npm start
```
* Validar o retorno do servidor rodando na porta definida

<img src="./assets/npm_start.png">


* Após validar o retorno de teste, digite 'Ctrl + C' para parar o servidor

### Configurar conexão com banco de dados
* Criar pasta 'config' dentro da pasta 'src'
* Criara arquivo 'db.js' na pasta 'config'
* Colar o código
```
const mysql = require('mysql2');
const dotenv = require('dotenv').config();

const connection = mysql.createConnection({
    host: process.env.DB_HOST,
    user: process.env.DB_USER,
    password: process.env.DB_PASSWORD,
    database: process.env.DB_DATABASE,
});

connection.connect( (err) => {
    if (err) {
        throw err;
    } else {        
        console.log("Mysql Connected!");
    }
});

module.exports = connection;
```

### Criar controller para cadastrar usuário
* Criar pasta 'controllers' dentro da pasta 'src'
* Criara arquivo 'usersController.js' na pasta 'controllers'
* Colar o código
```
const connection = require('../config/db');
const bcrypt = require('bcrypt');

async function listUsers(request, response) {
    connection.query('SELECT * FROM users', (err, results) => {
        if (results) {
            response
                .status(200)
                .json({
                    success: true,
                    message: `Sucesso! Lista de usuários.`,
                    data: results
                });
        } else {
            response
                .status(400)
                .json({
                    success: false,
                    message: `Não foi possível realizar a remoção. Verifique os dados informados`,
                    query: err.sql,
                    sqlMessage: err.sqlMessage
                });
        }
    })
}

async function storeUser(request, response) {   
    const params = Array(
        request.body.name,
        request.body.email,
        bcrypt.hashSync(request.body.password, 10)
    );

    const query = 'INSERT INTO users(name,email,password) values(?,?,?);';

    connection.query(query, params, (err, results) => {
        if (results) {
            response
                .status(201)
                .json({
                    success: true,
                    message: `Sucesso! Usuário cadastrado.`,
                    data: results
                });
        } else {
            response
                .status(400)
                .json({
                    success: false,
                    message: `Não foi possível realizar a ação. Verifique os dados informados`,
                    query: err.sql,
                    sqlMessage: err.sqlMessage
                });
        }        
    })
}

module.exports = {
    listUsers,
    storeUser
}
```

### Criar rota para cadastrar usuário
* Criar pasta 'routes' dentro da pasta 'src'
* Criar arquivo 'usersRouter.js' na pasta 'routes'
* Colar o código
```
const { Router } = require('express');
const router = Router();
const { listUsers, storeUser } = require('../controllers/usersController')

router.get('/users', listUsers);
router.post('/user/create', storeUser);

module.exports = router;
```

### Configurar API para disponibilizar a rota criada
* Abrir o arquivo 'app.js' 
* Adicionar a importação do 'usersRouter.js' na linha 5
```
const userRouter = require('./routes/usersRouter');
```
* Habilitar a utilização, adicionando o código na linha 10
```
app.use('/api', userRouter);
```

### Testar servidor e conexão com banco de dados
* Executar o comando com o gitBash
```
npm start
```
* Validar o retorno do servidor rodando na porta definida e o MySql conectado

<img src="./assets/npm_start_mysql.png">

### Criar controller para logar
* Criara arquivo 'loginController.js' na pasta 'controllers'
* Colar o código
```
const connection = require('../config/db');
const bcrypt = require('bcrypt');
const jwt = require('jsonwebtoken');

async function login(request, response) {
    const query = "SELECT * FROM users WHERE `email` = ?";
    
    const params = Array(
        request.body.email
    );

    connection.query(query, params, (err, results) => {
        if (results.length > 0) {
            bcrypt.compare(request.body.password, results[0].password, (errSenha, resultsSenha) => {
                if (resultsSenha) {                        
                    const userData = results[0];
                    const userId   = userData.id;
                    const token    = jwt.sign(
                        { userId },
                        'token',
                        { expiresIn: 300 }
                    );
                    userData['token'] = token;

                    response
                        .status(200)
                        .json({
                            success: true,
                            message: `Sucesso! Usuário conectado.`,
                            data: userData
                        });
                } else {
                    response
                        .status(401)
                        .json({
                            success: false,
                            message: `E-mail ou senha inválidos`,
                            data: errSenha
                        });
                }
            });
        }   
    })
}

module.exports = {
    login
}
```

### Criar rota para logar
* Criar arquivo 'loginRouter.js' na pasta 'routes'
* Colar o código
```
const { Router } = require('express');
const router = Router();
const { login } = require('../controllers/loginController');

router.post('/auth/login', login);

module.exports = router;
```