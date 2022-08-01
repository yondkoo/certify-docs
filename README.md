# Certify -ийг өөрийн сервер дээр асаах зааварчилгаа

## Системийг асаахад шаардагдах хангамжууд:
    1. Docker
    2. NGINX
    3. SSL

Доорх зааврын дагуу системийг асаахад Ubuntu серверийг ашигласан.

### Docker суулгах
1. apt -г шинэчилж, Docker програмыг суулгахад хэрэглэгдэх програмуудыг суулгана.

```shell
sudo apt-get update

sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```

2. Албан ёсны GPG түлхүүрийг нэмэх, програмын тогтвортой хувилбарыг татах санг нэмэх.

```shell
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

3. Дээрх командуудыг ажиллуулсны дараа таны компьютерт Docker програм суух орчин бүрдэнэ. Доорх командаар Docker болон түүнтэй холбоотой ажиллах програмуудыг суулгана.

```shell
sudo apt-get update

sudo apt-get install docker-ce docker-ce-cli containerd.io
```

4. Docker програм зөв суусан эсэхийг шалгаж hello-world-ийг ажиллуулъя.

```shell
docker run hello-world
``` 

5. Docker-compose програмыг суулгах

```shell
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

6. Татсан програмыг ажиллуулах эрхийг олгох.

```shell
sudo chmod +x /usr/local/bin/docker-compose
```
### Docker-compose ашиглан certify -ийг асаах
Certify -ийг асааж ашиглах docker image -үүд [энд](https://hub.docker.com/u/corexchain) байгаа.

    1. corexchain/certify-admin-app (admin web)
    2. corexchain/certify-main-app (customer web)
    3. corexchain/certify-api (backend service)

Дээрх гурван docker image -ийг доорх зааварт асаана.

docker-comopse.yaml файл:
```yaml
version: "3.0"

services:
  mysql-db:
    container_name: "mysql-db"
    image: mysql:5.7
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: <<Өгөгдлийн сангийн root хэрэглэгчийн нууц үг>>
    volumes:
      - "/<<Өгөгдийн сангийн файлуудыг Docker container -тэй холбох зам>>:/var/lib/mysql"
    networks:
      certify-vpc:
        ipv4_address: 10.10.0.9

  certify-api:
    container_name: "certify-api"
    image: corexchain/certify-api
    restart: always
    ports:
      - 1010:1010
    volumes:
      - "<<Файл хадгалах зам>>:/code/files_dir"
      - "<<Хаяг болон түлхүүрүүдийг хадгалах зам>>:/code/keystore"
      - "./static-files/static:/code/auth/static" # Энэ нь backend -ийн удирдах хэсэгт шаардлагатай учир өөрчлөх шаардлаггүй.
    env_file:
      - ./certify-api.env # certify-api -ийг асаахад шаардлагатай орчны утгуудыг оруулна.
    networks:
      certify-vpc:
        ipv4_address: 10.10.0.10

  certify-admin:
    container_name: "certify-admin-app"
    image: corexchain/certify-admin-app
    restart: always
    volumes:
      - "./certify-admin.js:/usr/share/nginx/html/configs/env.js" # certify-admin.js файлд certify-admin-app -ийн орчны утгуудыг оруулна.
    ports:
      - 1111:80
    networks:
      certify-vpc:
        ipv4_address: 10.10.0.11

  certify-main:
    container_name: "certify-main-app"
    image: corexchain/certify-main-app
    restart: always
    ports:
      - 1212:80
    volumes:
      - "./certify-main.js:/usr/share/nginx/html/config/env.js" # certify-main.js файлд certify-main-app -ийн орчны утгуудыг оруулна.
    networks:
      certify-vpc:
        ipv4_address: 10.10.0.12

networks:
  certify-vpc:
    ipam:
      driver: default
      config:
        - subnet: 10.10.0.0/16
          gateway: 10.10.0.1
```

Энэ тохиргооны файлд тохируулж буй замуудад өгөгдлийн сангийн өгөгдлүүд, блочкэйн дээрх хаягийн мэдээллүүд гэх зэрэг эмзэг мэдээллүүд байгаа тул нягтлаж, өөр газруудад backup хэлбэрээр хадгалах хэрэгтэй.

certify-api.env
```env
# DJANGO
DEBUG=False
SECRET_KEY=
MAX_FILE_SIZE=10000000

# DB
DB_NAME=<<Өгөгдлийн сангийн нэр>>
DB_USER=<<Өгөгдлийн сангийн хэрэглэгч>>
DB_PASSWORD=<<Өгөгдлийн сангийн хэрэглэгчийн нууц үг≥>>
DB_HOST=<<Өгөгдлийн сангийн хаяг>>
DB_PORT=<<Өгөгдлийн сангийн port>>

# Google
STORAGE_TYPE=local
STORAGE_LOCAL_PATH='/code/files_dir' # Файл хадгалах зам, энэ зам нь docker-compose дээрх файл хадгалах замтай таарч байх ёстой.

# Blockchain
TESTNET=True # Хэрэв Testnet бол True үгүй бол False, үүнээс шалтгаалж COREXCHAIN_NODE_URL болон CERTIFY_CONTRACT_ADDRESS -ийн утгуудыг солих ёстой.
KEY_STORE_PATH=/code/keystore
COREXCHAIN_NODE_URL=https://node-testnet.corexchain.io/
CERTIFY_CONTRACT_ADDRESS=0xCc546a88Db1aF7d250a2F20Dee42eC436F99e075

# CORS & ALLOWED HOST & CSRF
ALLOWED_HOSTS=10.0.70.72,localhost,127.0.0.1,<<Таны домайн нэр>>
CSRF_TRUSTED_ORIGINS=<<Таны домайн нэр>> CORS_ORIGIN_WHITELIST=http://localhost:8080,<<Таны домайн нэр>>
CORS_ALLOWED_ORIGINS=http://localhost:8080,<<Таны домайн нэр>>
```

certify-main.js
```javascript
window.env = {
    isLoginAllowed: true,
    isOrgAllowed: false,

    headerHideItems: [],

    dashboardUrl: 'https://', // Админ вебийн замыг URL -ийг оруулах.

    REACT_APP_SERVICE_URI: 'https://explorer-testnet.corexchain.io/graphql',
    REACT_APP_SERVICE_NAME: 'crx',
    REACT_APP_CERTIFY_CONTRACT_ADDRESS: '',
    REACT_APP_VERIFIED_ISSUER_ADDRESS: '',
    REACT_APP_COREXCHAIN_NODE_URL: 'https://node-testnet.corexchain.io',


    REACT_APP_AVAILABLE_CONTRACT_ADDRESSES: ['',
    '',
    '', ''],
    REACT_APP_VERIFIED_ISSUER_ADDRESS: '',
    REACT_APP_VERIFIED_ISSUER_ADDRESS_TESTNET: '',
    REACT_APP_COREXCHAIN_NODE_URL_TESTNET: 'https://node-testnet.corexchain.io',
};
```

certify-admin.js
```javascript
window.env = {
  VUE_APP_BACKEND_URL: 'https://certify.num.edu.mn/service', //Certify-api -ийн URL -ийг оруулах
  VERIFY_URL: 'https://', //Үндсэн веб асах URL -ийг оруулах.
  headerHideItems: [],
};
```

Дээрх зарим утгуудыг манай талтай холбогдож авах учир хоосон орхисон байгаа.

1. docker-compose.yaml файл дээрх тохиргоогоор certify -ийг асаах.
```shell
docker compose up -d
```
2. Тохиргооны дагуу ассан шалгаж мөн системийн логийг хэвлэж харах.
```shell
docker compose ps
docker compose logs -f
```