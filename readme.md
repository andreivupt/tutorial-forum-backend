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

### Configurar API para disponibilizar a rota criada
* Abrir o arquivo 'app.js' 
* Adicionar a importação do 'loginRouter.js' abaixo da variável 'userRouter';
```
const loginRouter = require('./routes/loginRouter');
```
* Adicionar a utilização, adicionando o código abaixo da linha "app.use('/api, 'userRouter');"
```
app.use('/api', loginRouter);
```

### Criar controller para listar e registrar posts
* Criara arquivo 'postsController.js' na pasta 'controllers'
* Colar o código
```
const connection = require('../config/db');

async function listPosts(request, response) {
    const query = 'SELECT p.*, (SELECT count(r.post_id) FROM reactions r WHERE r.post_id = p.id ) as likes, ' + 
    ' (SELECT count(c.post_id) FROM comments c WHERE c.post_id = p.id ) as comments ' +
    ' FROM posts p ' + 
    ' GROUP BY p.id ORDER BY p.id DESC';

    connection.query(query, (err, results) => {        
        if (results) {
            response
                .status(200)
                .json({
                    success: true,
                    message: `Sucesso! Lista de posts.`,
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

async function storePost(request, response) {    
    const params = Array(
        request.body.post,
        request.body.userId,        
    );
    
    const query = 'INSERT INTO posts(description,user_id) values(?,?);';

    connection.query(query, params, (err, results) => {        
        if (results) {
            response
                .status(201)
                .json({
                    success: true,
                    message: `Sucesso! Post cadastrado.`,
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
    listPosts,
    storePost
}
```

### Criar rota para listar e registrar posts
* Criar arquivo 'postsRouter.js' na pasta 'routes'
* Colar o código
```
const { Router } = require('express');
const router = Router();
const { listPosts, storePost } = require('../controllers/postsController')

router.get('/posts', listPosts);
router.post('/post/create', storePost);

module.exports = router;
```

### Configurar API para disponibilizar a rota criada
* Abrir o arquivo 'app.js' 
* Adicionar a importação do 'postRouter.js' abaixo da variável 'loginRouter';
```
const postRouter = require('./routes/postsRouter');
```
* Adicionar a utilização, adicionando o código abaixo da linha "app.use('/api, 'loginRouter');"
```
app.use('/api', postRouter);
```

### Criar controller para listar e registrar comentarios
* Criara arquivo 'commentsController.js' na pasta 'controllers'
* Colar o código
```
const connection = require('../config/db');

async function commentsByPostId(request, response) {

    const query = 
    " SELECT u.name, c.id, c.description, DATE_FORMAT(c.created_at, '%d/%m/%Y %H:%i:%s') as data_criacao, c.user_id " +
    " FROM comments c, users u" +
    " WHERE c.post_id = ? and c.user_id = u.id ";

    const params = Array(
        request.params.post_id
    );
    
    connection.query(query, params, (err, results) => {
        if (results) {
            response
                .status(200)
                .json({
                    success: true,
                    message: `Sucesso! Lista de comentarios.`,
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

async function storeComment(request, response) {    
    const params = Array(
        request.body.description,
        request.body.idUser,        
        request.body.idPost
    );

    const query = 'INSERT INTO comments(description,user_id,post_id) values(?,?,?);';

    connection.query(query, params, (err, results) => {
        if (results) {
            response
                .status(201)
                .json({
                    success: true,
                    message: `Sucesso! Comentário cadastrado.`,
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
    commentsByPostId,
    storeComment
}
```

### Criar rota para listar e registrar comentários
* Criar arquivo 'commentsRouter.js' na pasta 'routes'
* Colar o código
```
const { Router } = require('express');
const router = Router();
const { commentsByPostId, storeComment  } = require('../controllers/commentsController');

router.get('/comments/:post_id', commentsByPostId);
router.post('/comment/create', storeComment);

module.exports = router;
```

### Configurar API para disponibilizar a rota criada
* Abrir o arquivo 'app.js' 
* Adicionar a importação do 'commentRouter.js' abaixo da variável 'postRouter';
```
const commentRouter   = require('./routes/commentsRouter');
```
* Adicionar a utilização, adicionando o código abaixo da linha "app.use('/api, 'postRouter');"
```
app.use('/api', commentRouter);
```

### Criar controller para listar e registrar reações
* Criara arquivo 'reactionsController.js' na pasta 'controllers'
* Colar o código
```
const connection = require('../config/db');

async function reactionsByPostId(request, response) {

    const query = "SELECT sum(case when type = 'LIKE' and post_id = ? then 1 else 0 end) AS likes, " +
    "sum(case when type = 'DESLIKE' and post_id = ? then 1 else 0 end) AS deslikes FROM reactions;";

    const params = Array(
        request.params.post_id,
        request.params.post_id
    );
    console.log(params)
    connection.query(query, params, (err, results) => {        
        if (results) {
            response
                .status(200)
                .json({
                    success: true,
                    message: `Sucesso! Reações por post.`,
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

async function storeReaction(request, response) {
        
    const params = Array(        
        request.body.postId,        
        request.body.comment_id ? request.body.comment_id : null,
        request.body.userId,        
    );

    const query = 'INSERT INTO reactions(post_id,comment_id,user_id) values(?,?,?);';

    connection.query(query, params, (err, results) => {
        if (results) {
            response
                .status(201)
                .json({
                    success: true,
                    message: `Sucesso! Reação cadastrada.`,
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
    storeReaction,
    reactionsByPostId
}
```

### Criar rota para gerenciar reações
* Criar arquivo 'reactionsRouter.js' na pasta 'routes'
* Colar o código
```
const { Router } = require('express');
const router = Router();
const { storeReaction, reactionsByPostId } = require('../controllers/reactionsController');
const { route } = require('./usersRouter');

router.post('/reaction/create', storeReaction);
router.get('/reactions_post/:post_id', reactionsByPostId);

module.exports = router;
```

### Configurar API para disponibilizar a rota criada
* Abrir o arquivo 'app.js' 
* Adicionar a importação do 'reactionRouter.js' abaixo da variável 'commentRouter';
```
const reactionRouter = require('./routes/reactionsRouter');
```
* Adicionar a utilização, adicionando o código abaixo da linha "app.use('/api, 'commentRouter');"
```
app.use('/api', reactionsRouter);
```

## Teste de enpoints com Insomnia

* Cadastro de usuário

<img src="./assets/post_user.png">

* Login

<img src="./assets/post_login.png">

* Cadastro de post

<img src="./assets/post_posts.png">

* Listar posts

<img src="./assets/get_posts.png">

* Cadastro de comentário

<img src="./assets/post_comment.png">

* Listar comentários

<img src="./assets/get_comments.png">

* Cadastrar reação

<img src="./assets/post_reaction.png">









