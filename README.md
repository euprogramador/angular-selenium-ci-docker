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

### Docker com selenium hub

Então podemos usar o docker para executar as instâcias do selenium hub e dos outros browsers.
As imagens podem ser conferidas em: https://hub.docker.com/u/selenium/

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










