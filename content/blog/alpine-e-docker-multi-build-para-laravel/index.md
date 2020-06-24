---
title: Alpine e Docker Multi Build para Laravel
date: "2020-06-24T19:43:30.284Z"
description: Atonomia de um Dockerfile para Laravel
---

Alpine Linux é uma distribuição minimalista com foco em segurança
e simplicidade. Ano passado trabalhei em um projeto que envolvia
analisar vulnerabilidades de uma aplicação antes da empresa
pagar por uma certificação de teste de penetração. Um potencial
cliente estava exigindo tal certificação antes de assinar um
contrato significante. Ao utilizar ferramentas como Trivy ou Clair,
as imagens oficials de PHP possuem em torno de 400 
potenciais vulnerabilidades. Ao invés de tentar lidar com reduzir
a zona de ataque, migrar para o alpine parecia uma estratégia mais
eficiente.

#### Dockerfile

O Dockerfile que utilizamos têm as seguintes instruções:

```dockerfile
ARG COMPOSER_AUTH

#BASE
FROM alpine:3.12 as base

RUN apk update \
&&  apk add php7-pdo_mysql php7-json php7-tokenizer php7-mbstring php7-iconv php7-session php7-redis php7-fileinfo \
    php7-bcmath php7-simplexml php7-dom

#DEPENDENCIES
FROM base as dependencies

ARG COMPOSER_AUTH

COPY docker /

COPY . /app

WORKDIR /app

RUN apk add composer php7-curl php7-zip \
&&  rm /etc/php7/conf.d/00_opcache.ini \
&&  composer global require hirak/prestissimo \
&&  composer install --no-dev --no-interaction

#CLI
FROM dependencies as cli 

RUN apk add curl php7-xml php7-xmlwriter \
&&  composer install

#APP
FROM base

RUN apk add php7 php7-apache2 php7-opcache
 
COPY docker /

COPY --from=dependencies /app /app

RUN chown -R apache:apache /app/storage /app/bootstrap/cache \
&&  sed -ri -e 's!^(\s*ServerTokens)\s+\S+!\1 Prod!g' "/etc/apache2/httpd.conf" \
&&  sed -ri -e 's!^(\s*ServerSignature)\s+\S+!\1 Off!g' "/etc/apache2/httpd.conf" \
&&  sed -ri -e 's!^(\s*CustomLog)\s+\S+!\1 /proc/self/fd/1!g' "/etc/apache2/httpd.conf" \
&&  sed -ri -e 's!^(\s*ErrorLog)\s+\S+!\1 /proc/self/fd/2!g' "/etc/apache2/httpd.conf" \
&&  sed -i 's,#\(LoadModule rewrite_module modules/mod_rewrite.so\),\1,g' "/etc/apache2/httpd.conf"

EXPOSE 80

CMD ["httpd", "-DFOREGROUND"]
```

O argumento `COMPOSER_AUTH` é utilizado para instruir o composer
a autenticar com o Github para instalação de pacotes privados
desenvolvidos internamente na empresa. A camada `base` possui
a maioria das dependências necessárias para executar o código,
incluindo principalmente extensões PHP utilizadas pelo Laravel.

A camada `dependencies` já possui ferramentas adicionais tais como
composer e php7-zip que não serão incluídas na camada final que
rodará o sistema em produção. Trata-se de uma camada intermediária
que trará todo o conteúdo da pasta vendor necessária para o funcionamento
do sistema.

Em seguida há uma outra camada intermediária chamada `cli` onde
o `composer` é executado, mas dessa vez sem excluir as dependências
de desenvolvimento. Essa camada é muito útil para rodar comandos
como `artisan` ou `phpunit`. Vale lembrar que podemos construir
uma imagem dedicada para uma camada específica usando o argumento
`--target` como `docker build . -t app:cli --target cli`. 

Note que a camada final não possui um _alias_. Ela é a camada
que será construída quando o argumento `--target` não for
especificado. Nela, temos acesso aos arquivos das camadas
intermediárias bem como podemos começar a partir de um
ponto específico, por exemplo começar da camada `base` cujo não
possui composer instalado. Usando o `--from`, podemos copiar
os arquivos da camada `dependencies` que inclui toda a pasta
`vendor` mas que não inclui nenhum conteúdo desnecessário para
o funcionamento do sistema em produção.

Bem ao final da última camada há uma configuração para o Apache
não emitir assinatura do servidor, reduzindo a capacidade de
bots que vasculham a internet por vulnerabilidades de poder executar
um _targetered attack_ contra nossos servidores. As diretrizes
de CustomLog e ErrorLog são redirecionadas para stdout e stderr
uma vez que o Docker disponibilizará tais _streams_ diretamente
no terminal para podermos verificar o estado do container.

#### A pasta docker

Além do dockerfile em si, para construir a imagem com sucesso vamos
precisar de uma pasta `docker` na raíz do projeto. Ela é copiada
diretamente para `/` dentro do container. A ideia é poder replicar
completamente o sistema de arquivos e sobrescrever qualquer conteúdo
que seja relevante para o funcionamento do serviço, tal como 
as configurações do PHP e do Apache.

```ini
<VirtualHost *:80>
    DocumentRoot "/app/public"

    AllowEncodedSlashes NoDecode

    <Directory "/app/public">
        Options Indexes FollowSymlinks MultiViews
        AllowOverride All
        Require all granted
    </Directory>

    SetEnvIfNoCase X-Forwarded-Proto "^https$" HTTPS

    CustomLog /proc/self/fd/1 combined env=!DONT_LOG

    RewriteEngine On
    RewriteCond %{HTTP:X-Forwarded-Proto} =http
    RewriteRule . https://%{HTTP:Host}%{REQUEST_URI} [L,R=permanent]
</VirtualHost>
``` 

Esse arquivo de virtualização do Apache encontra-se em 
`docker/etc/apache2/conf.d/virtualhost.conf`. Quando o Docker
copia `docker` para `/`, o arquivo vai parar no destino desejado
`/etc/apache2/conf.d/virtualhost.conf`. Isso facilita o trabalho
em time e permite que o desenvolvedor saiba com certa facilidade
onde os arquivos do sistema operacional serão copiados. 

#### Conclusão

Com o docker multi stage build é possível reduzir ao máximo
os vestígios utilizados pela instalação de um sistema e reduzir
a superfície de ataque do container. Uma imagem construída pelas
instruções do Dockerfile especificado acima consegue facilmente
passar pelas verificações de vulnerabilidades do Trivy, do Clair
e do AWS ECR Vulnerability Scanner.

Se gostou desse artigo me segue no [Twitter](https://twitter.com/deleugyn)!
Se não gostou desse artigo me segue no [Twitter](https://twitter.com/deleugyn)
e me diga o porquê. 

Até a próxima!