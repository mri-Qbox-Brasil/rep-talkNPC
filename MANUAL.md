# rep-talkNPC — Manual

Framework de diálogo com NPCs: cria um ped com target, abre uma UI de conversa com câmera fixa no rosto e executa funções Lua a partir das opções escolhidas.

---

## Sumário

1. [Dependências](#dependências)
2. [Instalação](#instalação)
3. [Configuração](#configuração)
4. [Entrypoints para outros recursos](#entrypoints-para-outros-recursos)
5. [Estrutura de um NPC](#estrutura-de-um-npc)
6. [Estrutura de um diálogo](#estrutura-de-um-diálogo)
7. [Exemplo completo](#exemplo-completo)
8. [Debug](#debug)
9. [Estrutura de arquivos](#estrutura-de-arquivos)

---

## Dependências

| Recurso | Obrigatório | Observação |
|---|---|---|
| `ox_target` / `qb-target` / `qtarget` | Sim | Definido em `Config.Target`. É o único jeito de iniciar a conversa |

O recurso **não** usa framework (QBCore/ESX) em nenhum ponto do código — é agnóstico. Os NPCs, jobs e a lógica de negócio ficam por conta do recurso que o consome.

---

## Instalação

1. Copie a pasta `rep-talkNPC` para `resources/`.
2. Adicione ao `server.cfg`, **antes** dos recursos que forem criar NPCs:
   ```
   ensure rep-talkNPC
   ```
3. Ajuste `Config.Target` em `config.lua` para o target que você usa.
4. A pasta `web/build/` já vem compilada. Só rebuilde se for alterar a interface.

Não há SQL nem itens de inventário. Não há conflitos conhecidos.

> Este recurso não traz nenhum NPC pronto: `client/cl_ex.lua` está inteiramente comentado e serve só de exemplo. Sem outro recurso chamando `CreateNPC`, nada aparece no mundo.

---

## Configuração

Todas as opções ficam em `config.lua` (shared).

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `Config.Target` | string | Sim | Recurso de target usado. `'ox_target'` usa `addLocalEntity`; qualquer outro valor (`'qb-target'`, `'qtarget'`) usa `AddTargetEntity` |
| `Config.Talk` | string | Sim | Texto da opção de target. Recebe o nome do NPC via `%s` (padrão: `"Falar com %s"`) |
| `Config.Color.primaryColor` | string | Sim | Cor do hover dos botões da UI. Aceita cor Mantine (`"green"`, `"blue.7"`) ou hex |
| `Config.Color.secondaryColor` | string | Sim | Cor padrão da tag do NPC quando ela não é definida na criação. Mesmo formato acima |

As cores Mantine seguem o formato `{cor}.{tom}` — o catálogo está em https://mantine.dev/theming/colors/#default-colors.

---

## Entrypoints para outros recursos

### `CreateNPC(pedData, elements)`

Cria o ped, aplica animação/scenario, registra a opção de target e devolve o handle da entidade. O ped é deletado automaticamente quando o **recurso que o criou** for parado (`GetInvokingResource`).

```lua
local ped = exports['rep-talkNPC']:CreateNPC(pedData, elements)
```

### `changeDialog(mensagem, elements)`

Troca a mensagem **e** a lista de opções do diálogo aberto. Use para navegar entre "telas" da conversa.

```lua
exports['rep-talkNPC']:changeDialog('Temos trabalhos limpos e sujos. Qual você quer?', {
    [1] = { label = 'Limpos', shouldClose = true,  action = function() end },
    [2] = { label = 'Sujos',  shouldClose = true,  action = function() end },
})
```

### `updateMessage(mensagem)`

Troca **apenas** a fala do NPC, mantendo as opções atuais na tela.

```lua
exports['rep-talkNPC']:updateMessage('É uma equipe que faz scripts para FiveM.')
```

### Evento `rep-talkNPC:client:close`

Fecha a UI de diálogo, destrói a câmera e devolve o foco ao jogador.

```lua
TriggerEvent('rep-talkNPC:client:close')
```

---

## Estrutura de um NPC

Primeiro argumento de `CreateNPC`.

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `npc` | string | Sim | Nome do modelo do ped (ex.: `'u_m_y_abner'`) |
| `coords` | `vector4` / `vector3` | Sim | Posição do ped. O `w` (ou `heading`) define a direção |
| `heading` | number | Não | Heading aplicado após o spawn |
| `name` | string | Sim | Nome do NPC. Usado no target (`Config.Talk`) e no cabeçalho da UI |
| `tag` | string | Não | Etiqueta exibida ao lado do nome (ex.: `"Bot"`, cargo) |
| `color` | string | Não | Cor da tag. Se omitido, usa `Config.Color.secondaryColor` |
| `startMSG` | string | Não | Primeira fala do NPC ao abrir a conversa. Padrão: `'Hello'` |
| `animName` | string | Não | Dicionário de animação. Se definido, o ped toca `animDist` em loop |
| `animDist` | string | Não | Nome da animação dentro de `animName` |
| `animScenario` | string | Não | Scenario do GTA V (ex.: `'WORLD_HUMAN_CLIPBOARD'`). Usado **apenas se** `animName` não for definido |

O ped criado é invencível, congelado, imune a eventos temporários e olha para o jogador.

---

## Estrutura de um diálogo

Segundo argumento de `CreateNPC` (e de `changeDialog`). É um array indexado a partir de `1`.

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `label` | string | Sim | Texto do botão exibido ao jogador |
| `shouldClose` | bool | Sim | Se `true`, a UI fecha ao clicar na opção |
| `action` | function | Sim | Função executada no clique. É onde você chama `changeDialog`, `updateMessage` ou dispara os eventos do seu recurso |

Ao clicar, o NPC executa a animação de fala (`SetPedTalk`) antes de rodar a `action`.

A conversa fecha automaticamente se o jogador se afastar mais de **5 metros** do NPC (verificado a cada 500 ms).

---

## Exemplo completo

```lua
local coords = GetEntityCoords(PlayerPedId()) - vector3(0.5, 0.5, 1.0)

exports['rep-talkNPC']:CreateNPC({
    npc = 'u_m_y_abner',
    coords = vector4(coords.x, coords.y, coords.z, 0.0),
    name = 'Rep Scripts',
    animName = 'mini@strip_club@idles@bouncer@base',
    animDist = 'base',
    tag = 'Bot',
    color = 'blue.7',
    startMSG = 'Olá, eu sou o bot da Rep Scripts'
}, {
    [1] = {
        label = 'O que é a Rep Scripts?',
        shouldClose = false,
        action = function()
            exports['rep-talkNPC']:updateMessage('É uma equipe que faz scripts para FiveM.')
        end
    },
    [2] = {
        label = 'Que tipos de script vocês têm?',
        shouldClose = false,
        action = function()
            exports['rep-talkNPC']:changeDialog('Temos trabalhos limpos e sujos. Qual você quer?', {
                [1] = { label = 'Limpos', shouldClose = true, action = function() end },
                [2] = { label = 'Sujos',  shouldClose = true, action = function() end },
            })
        end
    },
    [3] = {
        label = 'Tchau',
        shouldClose = true,
        action = function()
            TriggerEvent('rep-talkNPC:client:close')
        end
    }
})
```

`client/cl_ex.lua` traz esse mesmo exemplo comentado, mais um exemplo maior de integração com um job.

---

## Debug

Logs de diagnóstico são controlados por convar, no formato `<nome-do-recurso>-debugMode`:

```
setr rep-talkNPC-debugMode 1
```

Com o valor `1`, a função `debugPrint` (em `client/utils.lua`) passa a imprimir no console do cliente.

---

## Estrutura de arquivos

```
rep-talkNPC/
├── client/
│   ├── cl_main.lua              — criação do ped, câmera, target, callbacks NUI e exports (CreateNPC, changeDialog, updateMessage)
│   ├── cl_ex.lua                — exemplos de uso, inteiramente comentados
│   └── utils.lua                — SendReactMessage e debugPrint (convar rep-talkNPC-debugMode)
├── config.lua                   — Config.Target, Config.Talk e cores da UI
├── web/
│   ├── build/                   — bundle compilado (ui_page do manifest)
│   ├── src/
│   │   ├── components/App.tsx   — UI do diálogo (fala do NPC, nome, tag, botões)
│   │   ├── providers/ConfigProvider.tsx — busca as cores via callback NUI getConfig
│   │   ├── hooks/useNuiEvent.ts — recebe show / changeDialog / updateMessage / close
│   │   ├── utils/fetchNui.ts    — callbacks NUI (click, close, getConfig)
│   │   ├── typings/configs.ts
│   │   └── theme/index.ts       — tema Mantine
│   └── package.json
└── fxmanifest.lua
```
