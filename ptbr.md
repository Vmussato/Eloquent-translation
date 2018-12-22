{{meta {load_files: ["code/chapter/07_robot.js", "code/animatevillage.js"], zip: html}}}

# Projeto: Um Robô

{{quote {author: "Edsger Dijkstra", title: "The Threats to Computing Science", chapter: true}

[...] the question of whether Machines Can Think [...] is about as
relevant as the question of whether Submarines Can Swim.

quote}}

{{index "artificial intelligence", "Dijkstra, Edsger", "project chapter", "reading code", "writing code"}}

Em capítulos de projeto eu vou parar de te falar de teoria por um breve momento e ao invés disso nós vamos trabalhar em um programa juntos. Teoria é necessária para aprender a programar, mas ler e entender programas de verdade é tão importante quanto.

Nosso projeto neste capítulo é de construir um autômato, um pequeno programa que execute uma tarefa em um mundo virtual. Nosso autômato vai ser um robô carteiro trazendo e levando pacotes.

## Meadowfield (Campo Prado)

{{index "roads array"}} 

A aldeia de Campo Prado não é muito grande. Ela consiste de onze lugares com quatorze estradas entre eles. Ela pode ser representada com este array de estradas:

```{includeCode: true}
const roads = [
  "Alice's House-Bob's House",   "Alice's House-Cabin",
  "Alice's House-Post Office",   "Bob's House-Town Hall",
  "Daria's House-Ernie's House", "Daria's House-Town Hall",
  "Ernie's House-Grete's House", "Grete's House-Farm",
  "Grete's House-Shop",          "Marketplace-Farm",
  "Marketplace-Post Office",     "Marketplace-Shop",
  "Marketplace-Town Hall",       "Shop-Town Hall"
];
```

{{figure {url: "img/village2x.png", alt: "The village of Meadowfield"}}}

A rede de estradas da aldeia forma um gráfico.  Um gráfico é uma coleção de pontos (lugares na aldeia) com linhas entre as (estradas). esse gráfico vai ser o mundo em que o nosso robô se move.

O array de strings não é muito fácil de se trabalhar. O que nós estamos interessados é nos destinos que podemos alcançar a partir de um determinado local. Vamos converter a lista de estradas para uma estrutura de dados que, para cada local, nos diga para onde podemos ir a partir dele.

{{index "roadGraph object"}}

```{includeCode: true}
function buildGraph(edges) {
  let graph = Object.create(null);
  function addEdge(from, to) {
    if (graph[from] == null) {
      graph[from] = [to];
    } else {
      graph[from].push(to);
    }
  }
  for (let [from, to] of edges.map(r => r.split("-"))) {
    addEdge(from, to);
    addEdge(to, from);
  }
  return graph;
}

const roadGraph = buildGraph(roads);
```

Dado um array de arestas `buildGraph` cria um objeto mapa que, para cada nó, armazena um array de nós conectados.

{{index "split method"}}

Ele usa o método `split` para ir da estrada de strings, que tem o formulario `"Start-End"`, para um array de dois elementos que contem o inicio e o fim como strings separadas.

## A tarefa

Nosso ((robo)) vai se mover através do vilarejo. Vai haver entregas em vários lugares, cada um endereçado para um outro lugar. O robo coleta entregas quando ele chega nelas, e as entrega quando chega nos seus destinos.


O automato precisa decidir, em cada ponto, para onde ir em seguida. Ele terminou seu trabalho quando cada pacote foi entregue.

{{index simulation, "virtual world"}}

Para ser capaz de simular este processo, precisamos definir um mundo virtual que sejamos capazes de descrever. Esse modelo nos mostra onde o robô está e onde os pacotes estão. Quando o robô decide se mover para algum lugar, precisamos atualizar o modelo que reflete esta nova situação.


Se você está pensando em termos de orientação a objeto, o seu primeiro impulso poderia ser o de definir objetos para todos os elementos deste mundo. Uma classe para o robô, uma para o pacote, talvez uma para os locais. Estes poderiam então armazenar propriedades que descrevam o seu estado atual, tais como a quantidade de pacotes para cada destino, que nós poderiamos mudar ao atualizar o mundo.

Isso está errado.

Na verdade, normalmente errado. O fato de que alguma coisa parece um objeto não significa automaticamente que ele deveria ser um objeto no seu programa. Escrever classes de maneira reflexiva para cada conceito na sua aplicação tende a gerar uma coleção de objetos interconectados cada um com o seu próprio estado internos. Programas assim são usualmente difíceis de entender e por consequência fáceis de quebrar.

Ao invés disso, vamos condensar a vila ((state)) em um conjunto básico de valores que a definem. Temos a localização atual do robô e a coleção de pacotes que não foram entregues, cada um destes pacotes possui uma localização e um endereço de destino. Apenas isso.

{{index "VillageState class", "persistent data structure"}}

Enquanto estamos fazendo isso, vamos garantir que não mudamos o seu estado enquanto o robô se move, mas na verdade criar um novo estado para a situação após a movimentação.

```{includeCode: true}
class VillageState {
  constructor(place, parcels) {
    this.place = place;
    this.parcels = parcels;
  }

  move(destination) {
    if (!roadGraph[this.place].includes(destination)) {
      return this;
    } else {
      let parcels = this.parcels.map(p => {
        if (p.place != this.place) return p;
        return {place: destination, address: p.address};
      }).filter(p => p.place != p.address);
      return new VillageState(destination, parcels);
    }
  }
}
```

O método `move` é onde a ação acontece. Primeiro ele verifica se existe uma estrada que vá do local atual para para o nosso destino, e em caso negativo, ele nos retorna o estado antigo, já que este não é um movimento válido. 

{{index "map method", "filter method"}}

Então, ele cria um novo estado com o destino do robô com esse lugar. Porém ele também precisa criar um novo conjunto de pacotes-pacotes que o robô está carregando(que estão no local atual do robô) e que precisam ser movidos junto com o robô para este novo local. E pacotes que estão endereçados para este novo local precisam ser entregues, ou seja, precisam ser removidos do conjunto de pacotes não entregues. A chamada para o `map` resolve o problema da movimentação e a chamada para o `filter` faz as entregas.

Os objetos pacotes não são alterados quando são movidos, mas na verdade são recriados. O método `move` cria uma nova vila no estado, mas deixa o antigo completamente intacto.

```
let first = new VillageState(
  "Post Office",
  [{place: "Post Office", address: "Alice's House"}]
);
let next = first.move("Alice's House");

console.log(next.place);
// → Alice's House
console.log(next.parcels);
// → []
console.log(first.place);
// → Post Office
```

O movimento faz com que os pacotes sejam entregues, e isso é refletido no estado seguinte. Mas o estado inicial descreve a situação onde o robô está no correio e o pacote ainda não foi entregue.

## Persistindo dados

{{index "persistent data structure", mutability, "data structure"}}

Estruturas de dados que não mudam são chamadas de _((imutáveis))_ ou _persistentes_.Elas  se comportam de maneira muito parecida com strings e números, eles são quem eles são, e permanecem dessa maneira, ao invés de conter coisas diferentes em momentos diferentes.

Em Javascript, praticamente tudo _pode_ ser alterado, portanto trabalhar com valores que devem ser persistentes exige um pouco de cuidado. Existe uma função chamada `Object.freeze`, que altera um objeto de maneira que tentativas de alterar suas propriedades são ignoradas. Você pode fazer uso dessa função para garantir que seus objetos não serão alterados, se você quiser ser cuidadoso. Congelar um objeto requer um trabalho extra do computador, e ignorar atualizações é algo que pode confundir e fazer alguém fazer a coisa errada. Então eu normalmente prefiro apenas dizer as pessoas que um determinado objeto não deve ser alterado, e eu espero que eles se lembrem disso.

```
let object = Object.freeze({value: 5});
object.value = 10;
console.log(object.value);
// → 5
```


Por que eu estou saindo do meu caminho para não mudar objetos, quando o idioma
está obviamente esperando por mim?

Porque isso me ajuda a entender meus programas. Isso é sobre complexidade
gestão novamente. Quando os objetos no meu sistema estão fixos, estáveis
coisas, posso considerar as operações nelas isoladamente - mudando para
A casa de Alice de um determinado estado inicial produz sempre o mesmo novo
Estado. Quando os objetos mudam com o tempo, isso adiciona uma nova dimensão
de complexidade para este tipo de raciocínio.

Para um sistema pequeno como o que estamos construindo neste capítulo, nós
poderia lidar com esse pouco de complexidade extra. Mas o mais importante
limite sobre o tipo de sistemas que podemos construir é o quanto podemos
Compreendo. Qualquer coisa que torne seu código mais fácil de entender
é possível construir um sistema mais ambicioso.

Infelizmente, apesar de entender um sistema baseado em dados persistentes
estruturas é mais fácil, _designing_ um, especialmente quando o seu
linguagem de programação não está ajudando, pode ser um pouco mais difícil. Apesar
vamos procurar oportunidades para usar estruturas de dados persistentes neste
livro, nós também estaremos usando os mutáveis.

## Simulação


Uma entrega ((robô)) olha para o mundo e decide em qual
direção que ele quer se mover. Como tal, poderíamos dizer que um robô é um
função que leva um objeto 'VillageState' e retorna o nome de um
lugar próximo.

{{index "runRobot function"}}

Porque queremos que os robôs consigam lembrar as coisas, para que eles possam
fazer e executar planos, também passamos a memória deles, e permitimos
para retornar uma nova memória. Assim, a coisa que um robô retorna é um objeto
contendo tanto a direção que deseja mover quanto um valor de memória
que será devolvido na próxima vez que for chamado.

```{includeCode: true}
function runRobot(state, robot, memory) {
  for (let turn = 0;; turn++) {
    if (state.parcels.length == 0) {
      console.log(`Done in ${turn} turns`);
      break;
    }
    let action = robot(state, memory);
    state = state.move(action.direction);
    memory = action.memory;
    console.log(`Moved to ${action.direction}`);
  }
}
```

Considere o que um robô precisa fazer para "resolver" um determinado estado. Deve escolher
todas as encomendas, visitando todos os locais que tem uma parcela, e
entregá-los, visitando todos os locais para os quais uma parcela é endereçada,
mas só depois de pegar o pacote.

Qual é a estratégia mais idiota que poderia funcionar? O robô poderia
apenas ande em uma direção aleatória a cada turno. Isso significa que, com
grande probabilidade, acabará por se encontrar em todas as parcelas e
também em algum momento chegar ao local onde eles devem ser entregues.

{{index "randomPick function", "randomRobot function"}}

Veja como isso poderia ser:

```{includeCode: true}
function randomPick(array) {
  let choice = Math.floor(Math.random() * array.length);
  return array[choice];
}

function randomRobot(state) {
  return {direction: randomPick(roadGraph[state.place])};
}
```

{{index "Math.random function", "Math.floor function", array}}

Lembre-se que `Math.random ()` retorna um número entre zero e um,
mas sempre abaixo de um. Multiplicando esse número pelo comprimento de um
array e, em seguida, aplicando `Math.floor` para nos dá um índice aleatório para
a matriz.

Como este robô não precisa se lembrar de nada, ele ignora sua
segundo argumento (lembre-se que funções JavaScript podem ser chamadas com
argumentos extras sem efeitos negativos) e omite a propriedade `memória`
em seu objeto retornado.

Para colocar este sofisticado robô para funcionar, primeiro precisamos de uma maneira de
crie um novo estado com algumas parcelas. Um método estático (escrito aqui por
diretamente adicionando uma propriedade ao construtor) é um bom lugar para colocar
essa funcionalidade.

```{includeCode: true}
VillageState.random = function(parcelCount = 5) {
  let parcels = [];
  for (let i = 0; i < parcelCount; i++) {
    let address = randomPick(Object.keys(roadGraph));
    let place;
    do {
      place = randomPick(Object.keys(roadGraph));
    } while (place == address);
    parcels.push({place, address});
  }
  return new VillageState("Post Office", parcels);
};
```

{{index "do loop"}}

Nós não queremos nenhum pacote enviado do mesmo lugar que eles
são endereçados para. Por esse motivo, o loop `do` continua escolhendo novos
coloca quando recebe um que é igual ao endereço.

Vamos começar um mundo virtual.

```{test: no}
runRobot(VillageState.random(), randomRobot);
// → Moved to Marketplace
// → Moved to Town Hall
// → …
// → Done in 63 turns
```


Leva o robô muitas voltas para entregar as parcelas, porque
não está planejando muito bem. Nós vamos resolver isso em breve.

{{if interactive

Para uma perspectiva mais agradável da simulação, você pode usar o
função `runRobotAnimation` que está disponível neste capítulo
ambiente de programação. Isto irá executar a simulação, mas em vez de
saída de texto, mostra o robô se movendo ao redor do mapa da aldeia.

```{test: no}
runRobotAnimation(VillageState.random(), randomRobot);
```

A maneira como o `runRobotAnimation` é implementado permanecerá um mistério para
agora, mas depois de ler os [capítulos posteriores] (dom) deste livro,
que discutem a integração do JavaScript em navegadores da Web, você poderá
para adivinhar como funciona.

if}}

## A rota do caminhão de correio

{{index "mailRoute array"}}

Devemos ser capazes de fazer muito melhor do que o aleatório ((robô)). A
melhoria fácil seria dar uma dica da maneira como o correio do mundo real
obras de entrega. Se encontrarmos uma rota que passe por todos os lugares no
aldeia, o robô poderia executar essa rota duas vezes, altura em que é
garantido para ser feito. Aqui está uma dessas rotas (a partir do post
escritório).

```{includeCode: true}
const mailRoute = [
  "Alice's House", "Cabin", "Alice's House", "Bob's House",
  "Town Hall", "Daria's House", "Ernie's House",
  "Grete's House", "Shop", "Grete's House", "Farm",
  "Marketplace", "Post Office"
];
```

{{index "routeRobot function"}}

Para implementar o robô que segue a rota, precisaremos usar
memória do robô. O robô mantém o resto de sua rota em sua memória, e
dropa o primeiro elemento a cada turno.

```{includeCode: true}
function routeRobot(state, memory) {
  if (memory.length == 0) {
    memory = mailRoute;
  }
  return {direction: memory[0], memory: memory.slice(1)};
}
```

Este robô já é muito mais rápido. Vai levar no máximo 26 turnos
(duas vezes a rota de 13 passos), mas geralmente menos.

{{if interactive

```{test: no}
runRobotAnimation(VillageState.random(), routeRobot, []);
```

if}}

## Encontrando o caminho

Ainda assim, eu realmente não ligaria cegamente seguindo uma rota fixa
comportamento inteligente. O ((robô)) poderia funcionar de forma mais eficiente se
ajustou seu comportamento ao trabalho real que precisa ser feito.

{{index pathfinding}}

Para fazer isso, tem que ser capaz de se mover deliberadamente em direção a um dado
parcela, ou para o local onde uma parcela tem que ser entregue.
Fazendo isso, mesmo quando o objetivo está a mais de um passo,
requer algum tipo de função de localização de rotas.

O problema de encontrar uma rota através de um ((gráfico)) é um
_ ((problema de pesquisa)) _. Podemos dizer se uma determinada solução (uma rota)
é uma solução válida, mas não podemos calcular diretamente a solução, o
forma como poderíamos para 2 + 2. Em vez disso, temos que continuar criando potencial
soluções até encontrarmos um que funcione.

Existe uma quantidade infinita de rotas possíveis através de um gráfico. Mas
Ao procurar uma rota de _A_ a _B_, estamos interessados ​​apenas em
os que começam em _A_. Nós também não nos importamos com rotas que visitam
o mesmo lugar duas vezes - essas definitivamente não são a rota mais eficiente
qualquer lugar. Então, isso reduz a quantidade de rotas que a rota
finder tem que considerar.

Na verdade, estamos mais interessados ​​na rota _shortest_. Então nós queremos
para nos certificarmos de que observamos rotas curtas antes de olharmos para as mais longas. UMA
boa abordagem seria "crescer" rotas a partir do ponto de partida,
explorando cada lugar acessível que ainda não tenha sido visitado, até
rota atinge o objetivo. Dessa forma, vamos explorar apenas as rotas que são
potencialmente interessante, e encontrar a rota mais curta (ou uma das
rotas mais curtas, se houver mais de um) para o objetivo.

{{index "findRoute function"}}

{{id findRoute}}

Esta é uma função que faz isso:

```{includeCode: true}
function findRoute(graph, from, to) {
  let work = [{at: from, route: []}];
  for (let i = 0; i < work.length; i++) {
    let {at, route} = work[i];
    for (let place of graph[at]) {
      if (place == to) return route.concat(place);
      if (!work.some(w => w.at == place)) {
        work.push({at: place, route: route.concat(place)});
      }
    }
  }
}
```

A exploração tem que ser feita na ordem certa - os lugares que foram
alcançado primeiro tem que ser explorado em primeiro lugar. Nós não podemos explorar imediatamente
um lugar assim que chegarmos, porque isso significaria lugares alcançados
de lá também seria explorado imediatamente, e assim por diante, mesmo
embora possa haver outros caminhos mais curtos que ainda não foram
explorado.

Portanto, a função mantém um _ ((lista de trabalho)) _. Esta é uma matriz de
lugares que devem ser explorados em seguida, junto com a rota que nos levou
lá. Começa apenas com a posição inicial e uma rota vazia.

A pesquisa, em seguida, opera tomando o próximo item na lista e
explorando isso, o que significa que todas as estradas que vão daquele lugar são
olhou para. Se um deles é o objetivo, uma rota acabada pode ser
retornou. Caso contrário, se não tivermos olhado para este lugar antes, um novo
item é adicionado à lista. Se nós olharmos para isso antes, já que nós
estão olhando rotas curtas primeiro, encontramos uma rota mais longa
para esse lugar ou um precisamente enquanto o existente, e nós
não precisa explorá-lo.

Você pode imaginar isso como uma teia de rotas conhecidas rastejando
desde o ponto de partida, crescendo uniformemente por todos os lados (mas nunca
emaranhado de volta em si mesmo). Assim que o primeiro fio atingir o
local objetivo, esse fio é rastreada até o início, dando-nos a nossa
rota.

{{index "connected graph"}}

Nosso código não lida com a situação em que não há mais trabalho
itens na lista de trabalho, porque sabemos que nosso gráfico é _conectado_,
o que significa que todos os locais podem ser acessados ​​de todos os outros locais.
Nós sempre seremos capazes de encontrar uma rota entre dois pontos, e o
pesquisa não pode falhar.

```{includeCode: true}
function goalOrientedRobot({place, parcels}, route) {
  if (route.length == 0) {
    let parcel = parcels[0];
    if (parcel.place != place) {
      route = findRoute(roadGraph, place, parcel.place);
    } else {
      route = findRoute(roadGraph, place, parcel.address);
    }
  }
  return {direction: route[0], memory: route.slice(1)};
}
```

{{index "goalOrientedRobot function"}}

Este robô usa seu valor de memória como uma lista de direções para mover,
assim como o robô que segue a rota. Sempre que essa lista está vazia,
tem que descobrir o que fazer a seguir. Leva o primeiro não entregue
parcela no conjunto e, se isso ainda não foi levantado, traça um
rota para isso. Se foi pego, ainda precisa ser
entregue, por isso cria uma rota para o endereço de entrega.

{{if interactive

Vamos ver como isso acontece.

```{test: no, startCode: true}
runRobotAnimation(VillageState.random(),
                  goalOrientedRobot, []);
```

if}}

Este robô geralmente termina a tarefa de entregar 5 parcelas em torno de
16 voltas. Um pouco melhor que o `routeRobot`, mas ainda assim definitivamente não
ótimo.

## Exercícios

### Medindo um robô

{{index "measuring a robot (exercise)", testing, automation, "compareRobots function"}}

É difícil comparar objetivamente ((robô)) s apenas deixando que eles resolvam
alguns cenários. Talvez um robô tenha conseguido tarefas mais fáceis ou
o tipo de tarefas em que é bom, enquanto o outro não.

Escreva uma função `compareRobots` que leva dois robôs (e seus
memória inicial). Deve gerar cem tarefas e deixar cada uma delas
os robôs resolvem cada uma dessas tarefas. Quando terminado, deve produzir o
número médio de etapas que cada robô executou por tarefa.

Por uma questão de justiça, certifique-se de dar cada tarefa para ambos
robôs, em vez de gerar tarefas diferentes por robô.

{{if interactive

```{test: no}
function compareRobots(robot1, memory1, robot2, memory2) {
  // Your code here
}

compareRobots(routeRobot, [], goalOrientedRobot, []);
```
if}}

{{hint

{{index "measuring a robot (exercise)", "runRobot function"}}

Você terá que escrever uma variante da função `runRobot` que,
em vez de registrar os eventos no console, retorna o número de
passos que o robô tomou para completar a tarefa.

Sua função de medição pode então, em um loop, gerar novos estados e
conte os passos de cada um dos robôs. Quando gerou o suficiente
medições, ele pode usar `console.log` para mostrar a média de cada
robot, que é a quantidade total de passos dados divididos pelo número
de medições.

hint}}

### Robot efficiency

{{index "robot efficiency (exercise)"}}

Você pode escrever um robô que conclua a tarefa de entrega mais rápido do que
`goalOrientedRobot`? Se você observar o comportamento desse robô,
coisas obviamente idiotas isso faz? Como isso poderia ser melhorado?

Se você resolveu o exercício anterior, talvez queira usar o seu
função `compareRobots` para verificar se você melhorou o robô.

{{if interactive

```{test: no}
// Your code here

runRobotAnimation(VillageState.random(), yourRobot, memory);
```

if}}

{{hint

{{index "robot efficiency (exercise)"}}

A principal limitação do `goalOrientedRobot` é que ele só considera
uma parcela de cada vez. Muitas vezes, ele vai de um lado para o outro
aldeia porque o pacote que acontece de estar olhando passa a ser
do outro lado do mapa, mesmo que haja outras muito mais próximas.

Uma solução possível seria calcular rotas para todos os pacotes e
então pegue o mais curto. Resultados ainda melhores podem ser obtidos, se
existem várias rotas mais curtas, preferindo as que vão para
pegar um pacote em vez de entregar um pacote.

hint}}

### Grupo persistente

{{index "persistent group (exercise)", "estrutura de dados persistente", "Set class", "set (estrutura de dados)", "Group class", "PGroup class"}}

A maioria das estruturas de dados fornecidas em um ambiente JavaScript padrão
não são muito adequados para uso persistente. Arrays têm `slice` e
métodos `concat`, que nos permitem criar facilmente novos arrays sem
danificando o antigo. Mas `Set`, por exemplo, não tem métodos para
criando um novo conjunto com um item adicionado ou removido.

Escreva uma nova classe `PGroup`, semelhante à classe` Group` do [Chapter
?] (object # groups), que armazena um conjunto de valores. Como `grupo`, tem
métodos `add`,` delete` e `has`.

Seu método `add`, no entanto, deve retornar uma instância _new_` PGroup`
com o membro fornecido adicionado e deixar o antigo inalterado.
Da mesma forma, `delete` cria uma nova instância sem um determinado membro.

A classe deve funcionar para chaves de qualquer tipo, não apenas strings. Faz
_not_ tem que ser eficiente quando usado com grandes quantidades de chaves.

O ((construtor)) não deve fazer parte da classe '((interface))
(embora você definitivamente deseje usá-lo internamente). Em vez disso,
é uma instância vazia, `PGroup.empty`, que pode ser usada como um começo
valor.

{{index singleton}}

Por que você só precisa de um valor `PGroup.empty`, em vez de ter um
função que cria um novo mapa vazio toda vez?

{{if interactive

```{test: no}
class PGroup {
  // Your code here
}

let a = PGroup.empty.add("a");
let ab = a.add("b");
let b = ab.delete("a");

console.log(b.has("b"));
// → true
console.log(a.has("b"));
// → false
console.log(b.has("a"));
// → false
```

if}}

{{hint

{{index "persistent map (exercise)", "Set class", array, "PGroup class"}}

A maneira mais conveniente de representar o conjunto de valores de membros permanece
ainda é um array, pois são fáceis de copiar.

{{index "método concat", "filter method"}}

Quando um valor é adicionado ao grupo, você pode criar um novo grupo com um
cópia da matriz original que possui o valor adicionado (por exemplo,
`concat`). Quando um valor é excluído, você o filtra da matriz.

A classe '((construtor)) pode pegar uma matriz como argumento e
armazene-a como a propriedade da instância (somente). Esse array nunca é
Atualizada.

{{index "static method"}}

Para adicionar uma propriedade (`empty`) a um construtor que não é um método, você
tem que adicioná-lo ao construtor após a definição da classe, como um
propriedade regular.

Você só precisa de uma instância `empty` porque todos os grupos vazios são
mesmo e instâncias da classe não mudam. Você pode criar muitos
diferentes grupos daquele único grupo vazio sem afetá-lo.

hint}}