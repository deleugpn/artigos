---
title: Rodando Laravel Dusk com Docker
date: "2020-06-24T18:48:30.284Z"
description: Usando docker-compose para rodar o Google Chrome
---

Laravel Dusk é um pacote muito bacana que o Taylor desenvolveu
para nos ajudar a rodar testes _end to end_ usando um navegador.
Recentemente eu estava trabalhando em um projeto que faz uso 
da sessão nativa do PHP ($_SESSION) e o phpunit não functiona
com o `session_start()` porque o PHP envia o PHPSESSID cookie
na `header` da resposta. A maior parte do sistema é testada
usando um mock na frente de uma classe que abstrai a sessão
do PHP, porém eu queria escrever pelo menos 1 teste que realmente
batesse na sessão para garantir que ela está funcionando.
Uma coisa que eu quis evitar nesse processo é de ter que instalar
o Google Chrome e todas as dependências necessárias para rodar
o _chrome driver_ no mesmo container que meu projeto roda, daí
surgiu a ideia de escrever um container dedicado para o Google
Chrome.

#### Dockerfile

A primeira coisa que fiz foi escrever um Dockerfile para o container
que vai ser responsável por manter o Chromedriver como um serviço.

```dockerfile
FROM alpine:3.12

RUN apk update && apk upgrade \
    && echo @edge http://nl.alpinelinux.org/alpine/edge/community >> /etc/apk/repositories \
    && echo @edge http://nl.alpinelinux.org/alpine/edge/main >> /etc/apk/repositories \
    && apk add --no-cache \
    chromium@edge \
    chromium-chromedriver@edge \
    xvfb \
    nss@edge \
    && rm -rf /var/lib/apt/lists/* \
    /var/cache/apk/* \
    /usr/share/man \
    /tmp/*

COPY ./chrome.sh /root/init.sh

RUN chmod +x /root/init.sh

RUN ln -s /usr/bin/chromium-browser /usr/bin/google-chrome

CMD ["/root/init.sh"]
```

Esse Dockerfile prepara o alpine para instalar o Google Chrome,
o Chromedriver e o xvfb. O arquivo `chrome.sh` que é copiado para
dentro do container é responsável por inicializar o Chromedriver.

```shell script
#!/usr/bin/env sh

Xvfb :99 &

/usr/bin/chromedriver --whitelisted-ips 0.0.0.0
```

#### Orquestrando os containers com Docker Compose

Depois de executar `docker build` para o Dockerfile acima, montei
um docker-compose para orquestrar os containers: application,
chrome e redis.

```yaml
version: '3.7'

services:
  cli:
    image: deleu/app-cli
    command: tail -f /dev/null
    volumes:
      - .:/app
    environment:
      - APP_ENV=testing

  web:
    image: deleu/app
    volumes:
      - .:/app
    environment:
      - APP_ENV=testing

  chrome:
    image: deleu/chrome
    environment:
      - DISPLAY=:99
```

O serviço identificado como `chrome` vai ligar o Dockerfile especificado
acima. Os containers `web` e `cli` fazem parte da minha aplicação.
O Laravel Dusk é inicializado pela linha de comando 
(`php artisan dusk`), mas ao utilizar o `$browser->visit()`, ele
instrui o navegador a acessar um site que, por sua vez, vai precisar
de um servidor web rodando, portanto preparei também o container
`web` com o Apache.

#### DuskTestCase

A classe DuskTestCase nos permite configurar como o Laravel Dusk
irá interagir com o Chromedriver. O método `driver()` deve apontar
para o container rodando o Chrome ao invés de apontar para o 
localhost.

```php
    protected function driver()
    {
        $options = (new ChromeOptions)->addArguments([
            '--disable-gpu',
            '--headless',
            '--no-sandbox',
            '--window-size=1920,1080',
        ]);

        return RemoteWebDriver::create(
            'http://chrome:9515', DesiredCapabilities::chrome()->setCapability(
                ChromeOptions::CAPABILITY, $options
            )
        );
    }
```

#### Browser

No código de teste, podemos instruir o navegador a usar o container
que está rodando o servidor web.

```php
    public function testBasicExample()
    {
        $this->browse(function (Browser $browser) {
            $browser->visit('http://web/')
                    ->assertSee('Laravel');
        });
    }

```

Uma outra forma de instruir o navegador a utilizar o endereço
do container web é através da variável de ambiente APP_URL. Com
ela é possível evitar ter que informar o protocolo e o domínio
em todos os testes.

#### Conclusão

O Chromedriver é um serviço e como tal podemos aproveitar os
mecanismos de comunicação entre containres do docker-compose
para não ter que instalar dependências desnecessárias em um
único container.

Se gostou desse artigo me segue no [Twitter](https://twitter.com/deleugyn)!
Se não gostou desse artigo me segue no [Twitter](https://twitter.com/deleugyn)
e me diga o porquê. 

Até a próxima!