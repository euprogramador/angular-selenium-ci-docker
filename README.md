# angular-selenium-ci-docker
Este repositório demonstra como usar o angular juntamente com o selenium hub para testes de unidade e aceitação em ambientes standalone e continuous integration/delivery

## Introdução

Testes são uma forma de garantir que a codificação do programa especificado, esteja em conformidade com o que foi solicitado.
Entretanto o benefício real de uma disciplina de desenvolvimento guiado por testes é realmente vista ao se fazer alterações futuras. Quando um teste acusa um erro, em que você programou, não ter que ficar depurando para achar a origem dos problemas é incrível. Quando corrigido, ter a garantia de que o que foi implementado e TESTADO, continua funcionando é muito bom. 

Mas para que tenhamos esta certeza é necessário força de vontade, montar ambientes que permitam esta prática nem sempre é fácil. A escrita destes tipos de testes também tem seus percalços.

Este pequeno guia fala da montagem de um ambiente que pode ser utilizado para o desenvolvimento de sistemas usando uma abordagem dirigida para a integração contínua.

## Phamtomjs

Phamtomjs é uma ferramenta incrível. A idéia de um browser sem UI e com uma api especificada acoplada é muito interessante. è fácil de configurar e, as diversas ferramentas de testes apoiam o seu uso. Entretanto phamtomjs tem problemas que podem nos levar a pensar se ele é uma boa escolha para o uso em testes, [Cris Le apontou em seu blog em 2013 do porque não usar phatomjs](http://www.chrisle.me/2013/08/5-reasons-i-chose-selenium-over-phantomjs/). 

Basicamente:

* Usa QtWebkit
* Não tem como inspecionar os elementos

Para isso é melhor usar um browser que o usuário usa. Podemos usar o chrome, firefox, IE ou safari. O IE e o safari não permitem o uso em containeres de forma facilitada, ou pelo menos não conheço ainda.

## Selenium HUB

Uma idéia interessante é usar o selenium hub e fazer com que os projetos apontem para o hub, no hub configuramos as instâncias de browser.

<pre>
+-----------------+           +--------------+                 +------------+
| suite de testes |           |              |===============> | instancia  |
| de unidade ou   |=========> | Hub selenium |                 | do chrome  |   
| integração      |           |              |                 +------------+           
+-----------------+           +--------------+ 
                                     ||
                                     =========================> +--------------+
                                                                |  instancia   |
                                                                |  dos outros  |
                                                                |  browsers    |
                                                                +--------------+   
</pre>

### Docker com selenium

Então podemos usar o docker para executar as instâcias do selenium hub e dos outros browsers.
As imagens podem ser conferidas em: https://hub.docker.com/u/selenium/

#### Docker com selenium hub e multiplas instancias de browser

Para iniciar o grid via docker: 
```bash
docker run -d -p 4444:4444 --name selenium-hub selenium/hub
```
Com isso temos um hub inicializado. Agora vamos inicializar os nodes que serão conetados ao hub para oferecer instancias para executar os testes.

Para disponibilizar uma instância de browser chrome use o comando docker abaixo:
```bash
docker run -d --link selenium-hub:hub selenium/node-chrome
```
Para disponibilizar uma instância de browser firefox use o comando docker abaixo:
```bash
docker run -d --link selenium-hub:hub selenium/node-firefox
```

#### Docker com selenium standalone

As instâncias standalone são instâncias que não precisam do hub para funcionar, elas já incorporam o browser e o hub juntos em uma única imagem, isso permite mais agilidade para um teste local.

Para executar uma instância standalone use:

```bash
docker run -d -p 4444:4444 selenium/standalone-chrome
```

## Angular

Este tutorial esta usando o angular cli para configurar um ambiente e ajustar a execução de um teste de unidade e integração usando o selenium, para isso usaremos a versão standalone que executa os testes em uma instancia especifica do browser.

O projeto deve ser inicializado normalmente usando o angular cli mais recente, no exemplo usamos o resolvedor de pacotes yarn, mas os mesmos comandos servem para o npm também.

```bash
ng new projeto-selenium
```
Com isso teremos uma pasta chamada `projeto-selenium`, que contém o projeto.

angular cli já inicia um ambiente com o karma e protractor configurados para um ambiente que temos uma GUI, como o nosso objetivo será de automátizar a execução em um ambiente de nuvem, esta abordagem não é apropriada para ser executada, uma vez que na nuvem, não teremos uma GUI. A abordagem aqui sugerida rodou perfeitamente usando gitlab pipelines e bitbucket pipelines, necessitando de poucos ajustes, que abordaremos mais a frente.

Uma vez com o projeto inicializado, precisamos ajustar o karma e o protractor para usar o selenium que foi inicializado anteriormente.

Para isso ajustamos o arquivo karma.conf.js e adicionar o plugin karma-webdriver-launcher:

```javascript
module.exports = function (config) {
  config.set({
    basePath: '',
    frameworks: ['jasmine', '@angular/cli'],
    plugins: [
      require('karma-jasmine'),
      require('karma-webdriver-launcher'), //// aqui substituimos o karma-chrome-launcher pelo karma-webdriver-launcher
      require('karma-jasmine-html-reporter'),
      require('karma-coverage-istanbul-reporter'),
      require('@angular/cli/plugins/karma')
    ],
    ....
```
Agora precisamos informar para o plugin onde está localizado o nosso selenium, fazemos isso usando uma configuração de browser customizada e adicionamos esta configuração no karma.conf.js:

```javascript
...
customLaunchers: {
      swd_chrome: {
         base: 'WebDriver',
          config: {
            hostname: '127.0.0.1' // endereço do host local pois fizemos bind de portas 4444 na imagem docker do selenium
          },
          browserName: 'chrome',
          version: '',
          name: 'Chrome',
          pseudoActivityInterval: 30000
      },
    },
....

browsers: ['swd_chrome'],

....
```
Também ajustamos para o report dos testes serem feitos no console. 

Precisamos também adicionar a dependência do webdriver e reporter no package.json:
```bash
yarn add karma-webdriver-launcher karma-verbose-reporter --dev
```
ou 
```bash
npm i karma-webdriver-launcher karma-verbose-reporter --save-dev
```

Segue abaixo o karma.conf.js completo com os ajustes usados:
```javascript
// Karma configuration file, see link for more information
// https://karma-runner.github.io/0.13/config/configuration-file.html


module.exports = function (config) {
  config.set({
    basePath: '',
    frameworks: ['jasmine', '@angular/cli'],
    plugins: [
      require('karma-jasmine'),
      require('karma-webdriver-launcher'),
      require('karma-jasmine-html-reporter'),
      require('karma-verbose-reporter'),
      require('karma-coverage-istanbul-reporter'),
      require('@angular/cli/plugins/karma')
    ],
    customLaunchers: {
      swd_chrome: {
         base: 'WebDriver',
          config: {
            hostname: '127.0.0.1'
          },
          browserName: 'chrome',
          version: '',
          name: 'Chrome',
          pseudoActivityInterval: 30000
      },
    },
    client:{
      clearContext: false, // leave Jasmine Spec Runner output visible in browser
      captureConsole: true,
      mocha: {
        bail: true
      }
    },
    coverageIstanbulReporter: {
      reports: [ 'html', 'lcovonly' ],
      fixWebpackSourcePaths: true
    },
    angularCli: {
      environment: 'dev'
    },
    reporters: ['progress',  'kjhtml', 'verbose'],
    port: 9876,
    colors: true,
    logLevel: config.LOG_INFO,
    autoWatch: true,
    browsers: ['swd_chrome'],
    singleRun: false
  });
};
```

Para rodar o teste agora usamos o comando abaixo:
```bash
ng test --colors --hostname=172.17.0.1  --code-coverage
```
o IP 172.17.0.1 é o ip da instância local do docker, pode ser usado o seu ip real. O karma apenas precisa achar a instância do selenium em execução. Também estamos instruindo a gerar o report de cobertura de código.

ao executar este comando obtemos o seguinte console:

```bash
$ ng test --colors --hostname=172.17.0.1
The option '--hostname' is not registered with the test command. Run `ng test --help` for a list of supported options.
 10% building modules 1/1 modules 0 active16 08 2017 13:13:58.193:WARN [karma]: No captured browser, open http://172.17.0.1:9876/
16 08 2017 13:13:58.204:INFO [karma]: Karma v1.7.0 server started at http://0.0.0.0:9876/
16 08 2017 13:13:58.204:INFO [launcher]: Launching browser swd_chrome with unlimited concurrency
16 08 2017 13:13:58.209:INFO [launcher]: Starting browser chrome via Remote WebDriver
16 08 2017 13:14:04.815:WARN [karma]: No captured browser, open http://172.17.0.1:9876/ 
16 08 2017 13:14:05.100:INFO [Chrome 59.0.3071 (Linux 0.0.0)]: Connected on socket 6keTI5DY6-_6XjF8AAAA with id 57429342
Chrome 59.0.3071 (Linux 0.0.0): Executed 0 of 3 SUCCESS (0 secs / 0 secs)
Chrome 59.0.3071 (Linux 0.0.0): Executed 1 of 3 SUCCESS (0 secs / 0.178 secs)
Chrome 59.0.3071 (Linux 0.0.0): Executed 2 of 3 SUCCESS (0 secs / 0.242 secs)
Chrome 59.0.3071 (Linux 0.0.0): Executed 3 of 3 SUCCESS (0 secs / 0.276 secs)
Chrome 59.0.3071 (Linux 0.0.0): Executed 3 of 3 SUCCESS (0.296 secs / 0.276 secs)


Suites and tests results:

 - AppComponent :
   * should create the app : ok
   * should have as title 'app' : ok
   * should render title in a h1 tag : ok

Browser results:

 - Chrome 59.0.3071 (Linux 0.0.0): 3 tests
   - 3 ok

```
Desta forma o desenvolvimento dos testes pode continuar e ao alterar os arquivos o karma roda novamente os testes.

caso queira executar apenas uma única vez sem o watch dos recursos basta adicionar a flag --single-run:
```bash
ng test --colors --hostname=172.17.0.1 --single-run  --code-coverage
```




