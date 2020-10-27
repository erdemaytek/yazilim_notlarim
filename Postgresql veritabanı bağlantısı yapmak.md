Bu makalemizde node js ve postgresql kullanılarak RESTful API nasıl geliştirilir kısaca onu anlatmaya çalışacağım.

# Gereksinimler

-  [Node.js ve npm]([Node.js](https://nodejs.org/en/)) bilgisayarınızda kurulmuş ve kullanılabilir olmalıdır.
- JavaScript sözdizimi ve temelleri hakkında bilgi sahibi olmalısınız
- Komut satırıyla çalışma konusunda temel bilgiye sahip olmalısınız
- Temel SQL CRUD işlemlerini yapabilmelisiniz.

# Ön Bilgi

Apilerimizi Express kütüphanesini kullanarak geliştireceğiz. Veri tabanı bağlantısı için [node-postgres (pg)](https://node-postgres.com/) kütüphanesini kullanacağız. Aşağıdaki komut ile kütüphaneleri projemize ekleyelim

`npm i express pg`

Yukarıdaki komutu verdiğinizde `node_module` ve `package.json` dosyasına ilgili bağlantılar gelmiş olacaktır.

`index.js ` Server ayarları için, `database_connect_info.js` veri tabanı bağlantısı ayarları için, `queries.js`  dosyasınıda veri tabanına gönderilecek sorgular için oluşturulacaktır.

# Server Yapısının Oluşturulması

`index.js`

```javascript
const express = require('express') 
const bodyParser = require('body-parser')
const app = express()

app.use(bodyParser.json())
app.use(
 bodyParser.urlencoded({
 extended: true,
 })
)

app.listen(3000, () => console.log("Server çalıştı."));
```

Body-Parser modülü; gönderilen post datasını obje olarak yakalamamızı sağlayan bir modüldür

- **app.use(bodyParser.json())**  
  JSON veri tipinde gelecek olan dataların kullanılabilmesi için bu tanımlamanın yapılması gerekmektedir.
- **app.use(bodyParser.urlencoded({ extended: true }))**  
  Encode(kodlanmış/şifrelenmiş) edilmiş urller üzerinde Body-Parser’ı kullanmak istiyorsanız eğer “extended” özelliğine true değerini vermeniz gerekir.

# Veri Tabanı Bağlantısı

`database_connect_info.js` isminde dosya oluşturalım ve aşağıdaki gibi veri tabanı bağlantı bilgilerini girelim.

```javascript
const Pool = require('pg').Pool

const pool = new Pool({
  user: 'postgres', 
  host: 'localhost', 
  database: 'dbname',
  password: 'pass',
  port: 5432,
})

module.exports = pool 
```

Bağlantı havuzu (pool) oluşturmak için node-postgres modülünü kullandık. Bu şekilde, her sorgu yaptığımızda istemciyi açıp kapatmak zorunda kalmayacağız.

# Yönlendirmeleri Ayarlamak (Routes)

`index.js` dosyasında aşağıdaki görüldüğü gibi, altı yönlendirme için altı fonksiyon oluşturduk.  Fonksiyonlarımızı `queries.js` dosyası içerisinde dolduracağız.

```javascript
const express = require('express') 
const bodyParser = require('body-parser')
const app = express()
const db = require("./queries")


app.use(bodyParser.json())
app.use(
 bodyParser.urlencoded({
 extended: true,
 })
)

// YÖNLENDİRME TANIMLAMALARI
app.get('/users', db.getUsers) 
app.get('/users/:id', db.getUserById)
app.post('/users', db.createUser)
app.put('/users/:id', db.updateUser)
app.delete('/users/:id', db.deleteUser)


app.listen(3000, () => console.log("Server çalıştı."));
```

Şimdi bu fonksiyonları yazabilmek için `queries.js` isminde bir dosya oluşturalım ve aşağıdaki gibi veritabanı işlemlerini yazalım

# Tüm Kullanıcıları Getir (GET)

`queries.js` dosyasına veri tabanı işlemleri yapabilmek için oluşturduğumuz `database_connect_info.js` dosyasını ekliyoruz.

`/users` adresinden kullanıcıları alabilmek için `GET `isteğinde bulunacağız ve veri tabanına sorgu yazabilmek için `pool.query()` apisini kullanacağız. 

Yazım Formatı 1: `pool.query('Sql Sorgusu yazılacak', callback_fonksiyon_yazılacak )` Bu fonksiyondaki error ile hataları, results ile de işlem sonucuna dair verileri bize veriyor.

```javascript
 var pool = require("./database_connect_info") // Veritabanı bağlantısı yapabilmek için bu sayfaya ekledik.

 const getUsers = (request, response) => {
    pool.query('SELECT * FROM users ORDER BY id ASC', (error, results) => {
      if (error) {
        throw error
      }
      response.status(200).json(results.rows) // Sorgu sonucundaki tüm kayıtları rows içerisinde almış oluyoruz.
    })
  }
```

# Id Bilgisine Göre Kullanıcı Getir (GET)

```javascript
const getUserById = (request, response) => {
  const id = parseInt(request.params.id) // users/:id den gelen veriyi alıyoruz. 

  pool.query('SELECT * FROM users WHERE id = $1', [id], (error, results) => {
    if (error) {
      throw error
    }
    response.status(200).json(results.rows)
  })
}
```

Örneğin:  `/users/1` şeklinde gelen istekteki `1 `parametresini` request.params.id` ile alabiliyoruz. params'dan sonra yazdığımız `id `ismi `index.js `dosyasındaki `app.get('/users/:id', db.getUserById)` tanımladığımız yönlendirmedeki /users/:**<mark>id</mark>** den gelmektedir.

Yazım Formatı 2: `pool.query('Sql Sorgusu yazılacak',[parametre1, parametre2], callback_fonksiyon_yazılacak )`

Sorgumuza request olarak gelen `id `değerimizi verebilmek için sorguda `$1` şeklinde işaretci koydum.  daha sonra `$1` için değer atamasını da sonraki parametrede `[id]` şekinde yazarak gönderdim. `$1`deki 1 sayısı `[1.parametre]` demektir.

# Yeni Kullanıcı Ekle (POST)

`/users` adresine gelen `POST `işleminin kayıdını aşağıdaki gibi yaptık. Öncelikle `request.body` ile gönderilen değerleri yakaladık. daha sonra bunları sorgumuza parametre vererek gönderdik.

Soruguda kullandığımız` RETURNING *` ifadesi kayıt işlemi başarılı olursa aynı değerleri bize geri döndürmesini sağlaması amacıyla yazdım. Bu genelde kayıt edilen verinin tablodaki id değerini almak için kullanılır.

```javascript
 const createUser = (request, response) => {
    const { name, email } = request.body
    pool.query('INSERT INTO users (name, email) VALUES ($1, $2) RETURNING *', [name, email], (error, results) => {
      if (error) {
        throw error
      }
      response.status(201).send(`User added with ID: ${results.rows[0].name}`)
    })
  }
  })
}
```

# Mevcut Kullanıcıyı Güncelleme (PUT)

`/users/:id `adresine yapılan `put `isteğinde gelen kullanıcı id'sine göre veri tabanında bulunan name ve email alanlarını güncellemiş olduk.

```javascript
const updateUser = (request, response) => {
    const id = parseInt(request.params.id)
    const { name, email } = request.body

    pool.query(
      'UPDATE users SET name = $1, email = $2 WHERE id = $3 RETURNING *',
      [name, email, id],
      (error, results) => {
        if (error) {
          throw error
        }
        response.status(200).json({Mesaj:"Kayıt Güncellendi"})
      }
    )
  }
```

# Kullanıcı Silme

`/users/:id`adresine yapılan `DELETE`isteğinde gelen kullanıcı id'sine göre veri tabanında bulunan kaydı silmiş olduk.

```javascript
 const deleteUser = (request, response) => {
    const id = parseInt(request.params.id)

    pool.query('DELETE FROM users WHERE id = $1', [id], (error, results) => {
      if (error) {
        throw error
      }
      response.status(200).send(`Kullanıcı Silindi - ID: ${id}`)
    })
  }
```

### Dosyanın Son Hali ve Fonksiyonların Dışarı Çıkarılması

```javascript
var pool = require("./database_connect_info")

  const getUsers = (request, response) => {
    pool.query('SELECT * FROM users ORDER BY id ASC', (error, results) => {
      if (error) {
        throw error
      }
      response.status(200).json(results.rows)
    })
  }

  const getUserById = (request, response) => {
    const id = parseInt(request.params.id)

    pool.query('SELECT * FROM users WHERE id = $1', [id], (error, results) => {
      if (error) {
        throw error
      }
      response.status(200).json(results.rows)
    })
  }

  const createUser = (request, response) => {
    const { name, email } = request.body
    pool.query('INSERT INTO users (name, email) VALUES ($1, $2) RETURNING *', [name, email], (error, results) => {
      if (error) {
        throw error
      }
      console.log(results.rows[0])
      response.status(201).json({data:results.rows[0], info:"Kayıt Eklendi"})
    })
  }

  const updateUser = (request, response) => {
    const id = parseInt(request.params.id)
    const { name, email } = request.body

    pool.query(
      'UPDATE users SET name = $1, email = $2 WHERE id = $3 RETURNING *',
      [name, email, id],
      (error, results) => {
        if (error) {
          throw error
        }
        console.log(results.rows)
        response.status(200).json({data:results.rows[0], info:request.body})
      }
    )
  }

  const deleteUser = (request, response) => {
    const id = parseInt(request.params.id)

    pool.query('DELETE FROM users WHERE id = $1', [id], (error, results) => {
      if (error) {
        throw error
      }
      response.status(200).send(`Kullanıcı Silindi - ID: ${id}`)
    })
  }

  module.exports = {
    getUsers,
    getUserById,
    createUser,
    updateUser,
    deleteUser,
  }
```

`index js` dosyasından burada yazılan fonkisyonlara erişebilmek için`module.exports`ile fonksiyonları dışarı aktarıyoruz.
