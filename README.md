# angular-selenium-ci-docker
Este repositório demonstra como usar o angular juntamente com o selenium hub para testes de unidade e aceitação em ambientes standalone e continuous integration/delivery

[![Build Status](https://travis-ci.org/euprogramador/angular-selenium-ci-docker.svg?branch=master)](https://travis-ci.org/euprogramador/angular-selenium-ci-docker)

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

### angular + karma + selenium

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

#### karma em ambiente de CI

caso queira executar apenas uma única vez sem o watch dos recursos basta adicionar a flag --single-run:
```bash
ng test --colors --hostname=172.17.0.1 --single-run  --code-coverage
```

## Angular + protractor + selenium

Protractor é um framework usado para testes de aceitação. Ele já vem preconfigurado para a execução em uma máquina, entretanto para usar em um ambiente de integração contínua juntamente com o selenium precisamos de alguns ajutes:

o arquivo é o protractor.conf.js, que precisamos modificar.

A configuração é simples, basta adicionar a linha abaixo:
```javascript
seleniumAddress: 'http://127.0.0.1:4444/wd/hub',
```
Esta linha informa a localização do selenium para o protractor.

também devemos dizer para o protractor que não usaremos a api diretamente:
```javascript
directConnect: false,
```
Segue o arquivo completo protractor.conf.js:

```javascript
// Protractor configuration file, see link for more information
// https://github.com/angular/protractor/blob/master/lib/config.ts

const { SpecReporter } = require('jasmine-spec-reporter');

exports.config = {
  seleniumAddress: 'http://127.0.0.1:4444/wd/hub',
  allScriptsTimeout: 11000,
  specs: [
    './e2e/**/*.e2e-spec.ts'
  ],
  capabilities: {
    'browserName': 'chrome'
  },
  directConnect: false,
  baseUrl: 'http://localhost:4200/',
  framework: 'jasmine',
  jasmineNodeOpts: {
    showColors: true,
    defaultTimeoutInterval: 30000,
    print: function() {}
  },
  onPrepare() {
    require('ts-node').register({
      project: 'e2e/tsconfig.e2e.json'
    });
    jasmine.getEnv().addReporter(new SpecReporter({ spec: { displayStacktrace: true } }));
  }
};
```
Após isso basta chamar o teste e2e:

```bash
ng e2e --hostname=172.17.0.1
** NG Live Development Server is listening on 172.17.0.1:49152, open your browser on http://172.17.0.1:49152 **
Date: 2017-08-16T17:05:08.851Z                                                          
Hash: d88d4dc0d2a223e579d7
Time: 9101ms
chunk {inline} inline.bundle.js, inline.bundle.js.map (inline) 5.83 kB [entry] [rendered]
chunk {main} main.bundle.js, main.bundle.js.map (main) 2.28 MB {inline} [initial] [rendered]
chunk {polyfills} polyfills.bundle.js, polyfills.bundle.js.map (polyfills) 204 kB {inline} [initial] [rendered]
chunk {styles} styles.bundle.js, styles.bundle.js.map (styles) 11.3 kB {inline} [initial] [rendered]

webpack: Compiled successfully.
[14:05:09] I/update - chromedriver: file exists /tmp/projeto-selenium/node_modules/webdriver-manager/selenium/chromedriver_2.31.zip
[14:05:09] I/update - chromedriver: unzipping chromedriver_2.31.zip
[14:05:10] I/update - chromedriver: setting permissions to 0755 for /tmp/projeto-selenium/node_modules/webdriver-manager/selenium/chromedriver_2.31
[14:05:10] I/update - chromedriver: chromedriver_2.31 up to date
[14:05:10] I/launcher - Running 1 instances of WebDriver
[14:05:10] I/hosted - Using the selenium server at http://127.0.0.1:4444/wd/hub
Jasmine started

  projeto-selenium App
    ✓ should display welcome message

Executed 1 of 1 spec SUCCESS in 2 secs.
[14:05:14] I/launcher - 0 instance(s) of WebDriver still running
[14:05:14] I/launcher - chrome #01 passed
```
Com isso o protractor usou a instância do chrome do selenium.

## Pipelines

Pipelines podem ser ligados a um repo git fazendo com que a cada commit sejam executados diversos passos como testes, documentação, etc....

### Bitbucket

Bitbucket possui um conceito de pipelines embutido no código que é executado a cada commit podemos colocar estes comandos para usar o recurso de pipelines do bitbucket.

Para isso criamos um arquivo bitbucket-pipelines.yml:
```yml
image: netczuk/node-yarn:node-6-yarn-0.18.1
pipelines: 
  default: 
    - step: 
        caches:
          - node
        script: 
          - yarn 
          - ./node_modules/.bin/ng test --host $(hostname -i) --single-run --coverage
          - ./node_modules/.bin/ng e2e --host $(hostname -i)
        services: 
          - selenium 
          
          
definitions: 
  services: 
    selenium: 
      image: selenium/standalone-chrome
```

Com isso podemos usar os testes no bitcket e esta configuração executa os testes usando o browser chrome.

NOTA: o bitbucket tem uma quantidade de 50 minutos para uso gratuito por mês.

### Gitlab

O Gitlab também possui recursos de pipeline, atualmente é o que uso, fiz a instalação do gitlab com runner docker e habilitei a instalação do pipelines do gitlab.

para usar os testes de integração continua no gitlab, criei os arquivos karma.conf.ci.js e protractor.conf.ci.js, com configurações personalizadas para o ambiente de CI/CD do gitlab, isso foi necessário pois o host que roda o selenium precisa ser nomeado, para acessar o selenium, não podendo usar $(hostname -I).

karma.conf.ci.js:
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
            hostname: 'selenium-unit' // nome do service que será disponibilizado no gitlab pipelines
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
protractor.conf.ci.js
```javascript
// Protractor configuration file, see link for more information
// https://github.com/angular/protractor/blob/master/lib/config.ts

const { SpecReporter } = require('jasmine-spec-reporter');

exports.config = {
  seleniumAddress: 'http://selenium-e2e:4444/wd/hub', // endereço aqui mudou. será fornecido pelo gitlab pipelines
  allScriptsTimeout: 11000,
  specs: [
    './e2e/**/*.e2e-spec.ts'
  ],
  capabilities: {
    'browserName': 'chrome',

  },
  directConnect: false,
  baseUrl: 'http://localhost:4200/',
  framework: 'jasmine',
  jasmineNodeOpts: {
    showColors: true,
    defaultTimeoutInterval: 30000,
    print: function() {}
  },
  onPrepare() {
    require('ts-node').register({
      project: 'e2e/tsconfig.e2e.json'
    });
    jasmine.getEnv().addReporter(new SpecReporter({ spec: { displayStacktrace: true } }));
  }
};
```

agora criamos o arquivo .gitlab-ci.yml:
```javascript
image: netczuk/node-yarn:node-6-yarn-0.18.1
  
stages:
  - build
  - tests

download_dependencies:
  stage: build
  cache:
    key: "$CI_COMMIT_REF_NAME"
    untracked: true
    paths:
    - node_modules/
  script:
    - yarn

unit_tests:
  stage: tests
  services:
  - name: selenium/standalone-chrome
    alias: selenium-unit
  cache:
    key: "$CI_COMMIT_REF_NAME"
    untracked: true
    policy: pull
    paths:
    - node_modules/
  script:
  # necessário rodar este script antes pois ele não faz parte do processo de instalação
  # vide: https://github.com/karma-runner/karma-sauce-launcher/issues/117 (comentario voltrevo de 9 de abril)
    - node node_modules/wd/scripts/build-browser-scripts.js
    - ng test --config=karma.conf.ci.js --single-run --hostname=$(hostname -i) --code-coverage

tests_e2e:
  stage: tests
  services:
  - name: selenium/standalone-chrome
    alias: selenium-e2e
  cache:
    key: "$CI_COMMIT_REF_NAME"
    untracked: true
    policy: pull
    paths:
    - node_modules/
  script:
    - ng e2e --host $(hostname -i) --config protractor.conf.ci.js   
```





