# 🏠 00 — IP e Mask explicados do zero (a analogia da rua e da casa)

> Este doc existe para quem está a ver isto pela primeira vez na vida. Sem jargão, sem binário — só a lógica.

## A ideia central

Um IP é o "endereço" de um computador. Uma mask diz-te **quanto desse endereço é a "rua" e quanto é o "número da casa"**.

Dois computadores só conseguem falar diretamente um com o outro se estiverem **na mesma rua**.

## Como ler isto na prática

O IP e a mask têm sempre 4 números, um por baixo do outro, na mesma posição:

```
IP:   104 . 99 . 23 . 12
Mask: 255 . 255 . 255 . 0
```

A regra é: olha número a número, na **mesma posição**.

| Posição | IP | Mask | O que a mask diz |
|---|---|---|---|
| 1ª | `104` | `255` | faz parte da rua |
| 2ª | `99` | `255` | faz parte da rua |
| 3ª | `23` | `255` | faz parte da rua |
| 4ª | `12` | `0` | faz parte da casa |

- Onde a mask tem **255** → esse número é a **rua** (tem de ser igual para dois dispositivos se falarem)
- Onde a mask tem **0** → esse número é a **casa** (pode ser diferente, identifica cada dispositivo)

Não precisas de saber binário para perceber isto — pensa só assim: `255` é o número máximo possível, então a mask está a dizer "isto tem de bater 100%, sem margem". `0` é o mínimo, então é "isto pode ser qualquer coisa, não interessa".

### Exemplo aplicado

```
104.99.23.12   com mask   255.255.255.0
    ↓   ↓   ↓      ↓
   rua rua rua   casa
```

Rua: `104.99.23`
Casa: `12`

Qualquer outro dispositivo com IP `104.99.23.<qualquer coisa>` está na mesma rua e consegue falar diretamente com este.

## Como escolher o número da casa

Quando a mask tem `0` numa posição, tens liberdade — mas com 3 regras:

1. ❌ **Não pode ser `0`** — reservado, significa "a rua toda", não uma casa específica
2. ❌ **Não pode ser `255`** — reservado, significa "mandar para todas as casas ao mesmo tempo" (broadcast)
3. ❌ **Não pode repetir** um número já usado por outro dispositivo na mesma rua

Fora isso, **pode ser mesmo qualquer coisa simples** — `1`, `2`, `3`, `50`... não há "o número certo". É como escolher um número de porta livre.

## Terminologia (para o README oficial)

Isto que temos estado a fazer tem nome:

| Termo | O que é |
|---|---|
| **Topologia** | O desenho da rede — quem está ligado a quem (os computadores e linhas no ecrã) |
| **Endereçamento / IP addressing** | Escolher os números (IP + mask) certos para cada aparelho |
| **Subnetting** | A técnica de calcular os ranges certos para cada "rua" (rede) |

Não é "desenhar a rede" — o desenho já vem feito. O que fazemos é **atribuir endereços IP** dentro dessa topologia, para que os dispositivos consigam falar entre si.

## Exemplo completo (Nível 1 do NetPractice)

Situação: o teu PC (host A) precisa de falar com o computador do teu irmão (host B).

```
interface B1 (irmão, já fixo):
  IP:   104.99.23.12
  Mask: 255.255.255.0
  → rua: 104.99.23   |   casa: 12

interface A1 (o teu PC, a corrigir):
  Mask: 255.255.255.0   (já está certa — igual à do irmão)
  IP: precisa de ter a MESMA rua (104.99.23) e uma casa diferente de 12
```

✅ Solução válida: `104.99.23.1` (rua igual, casa diferente, nem `0` nem `255`)

## Resumo de uma linha

> Mesma rua (números onde a mask tem `255`) = conseguem falar. Casa (números onde a mask tem `0`) = qualquer número livre, exceto `0`, `255`, ou repetido.
