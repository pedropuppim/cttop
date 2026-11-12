# cttop

**Um `top` para containers do Proxmox que mostra só o que é seu.**

Monitor de terminal para CTs (LXC): CPUs realmente disponíveis, uso real do container, memória, swap e os principais processos. Um arquivo Python, zero dependências.

```
╭─ cttop · meu-container ───────────────────────────────────────────────────── 15:06:05 ─╮
│ CPU ■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■  37.4% │
│                                                          ▁▂▂▃▅▆▅▃▂▂▃▄▆▇▆▅▃▃▂▃▄▅▅▄▃▃▄▅▆ │
│ 24 ■■■■■■■■■■■■■■■■■■  52%  25 ■■■■■■■■■■■■■■■■■■  18%  26 ■■■■■■■■■■■■■■■■■■  61%     │
│ 54 ■■■■■■■■■■■■■■■■■■  19%                                                             │
│ 4 nucleos  ·  rodando 3  ·  espera por CPU 1.2%  ·  106 processos, 152 threads  ·  up  │
╰────────────────────────────────────────────────────────────────────────────────────────╯
╭─ memoria ──────────────────────────────────────────────────────────────────────────────╮
│ RAM  ■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■ 963M / 9.3G   10%  cache 2.3G  │
│ Swap sem swap                                                                          │
╰────────────────────────────────────────────────────────────────────────────────────────╯
╭─ processos ────────────────────────────────────────────── c/m ordenar · q sair · 1.5s ─╮
│     PID USUARIO     CPU%▼  MEM%      RES     TEMPO  COMANDO                            │
│    1658 www-data    21.3    0.4    36.0M   0:14:01  /usr/sbin/apache2 -D FOREGROUND    │
│    1282 mysql       12.8    4.6     430M   2:02:28  mysqld --default-time-zone=-03:00  │
│     304 root         0.4    0.4    39.7M   0:03:25  /usr/bin/containerd                │
╰────────────────────────────────────────────────────────────────────────────────────────╯
```

No terminal as barras têm gradiente verde → amarelo → vermelho.

## Instalação

Precisa só de Python 3, que já vem na maioria dos templates de CT. Nada de `pip`.

Baixar e colocar no PATH para todos os usuários:

```bash
sudo curl -fsSL https://raw.githubusercontent.com/pedropuppim/cttop/master/cttop -o /usr/local/bin/cttop
sudo chmod +x /usr/local/bin/cttop
```

Sem `sudo`, só para o seu usuário:

```bash
mkdir -p ~/.local/bin
curl -fsSL https://raw.githubusercontent.com/pedropuppim/cttop/master/cttop -o ~/.local/bin/cttop
chmod +x ~/.local/bin/cttop
```

Se depois disso o comando não for encontrado, `~/.local/bin` não está no seu PATH. Adicione e reabra o terminal:

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
```

Sem `curl` no CT, use `wget -qO` no lugar de `curl -fsSL ... -o`.

### Instalar em vários CTs de uma vez

A partir da sua máquina, com o repositório clonado:

```bash
for host in ct1.exemplo.com ct2.exemplo.com; do
  scp cttop "$host:/tmp/cttop"
  ssh "$host" 'sudo install -m 755 /tmp/cttop /usr/local/bin/cttop && rm /tmp/cttop'
done
```

## Por que isso existe

Eu tenho um servidor Proxmox com muitos núcleos e vários CTs. Cada CT recebe só alguns núcleos, mas o `htop` dentro dele lista **todos os CPUs do host** e marca quase todos como `offl`:

```
  0[offl]   4[offl]   8[offl]  12[offl]  16[offl]  20[offl]  24[0.0%]
  1[offl]   5[offl]   9[offl]  13[offl]  17[offl]  21[offl]  25[0.0%]
  2[offl]   6[offl]  10[offl]  14[offl]  18[offl]  22[offl]  26[0.0%]
  3[offl]   7[offl]  11[offl]  15[offl]  19[offl]  23[offl]  27[offl]
```

Metade da tela vira ruído e os poucos núcleos que importam se perdem no meio. A motivação do `cttop` foi essa: **esconder os processadores offline** e mostrar só os que o container pode usar.

No caminho, ele corrige outras duas coisas que o `htop` mostra errado dentro de um CT:

| No `htop` dentro do CT | No `cttop` |
|---|---|
| Lista todos os CPUs do host, quase todos `offl` | Só os núcleos liberados para o CT |
| Load average do host inteiro | Processos rodando e espera por CPU do próprio CT |
| `CPU%` de processo acima de 100% (ex.: 622%) | `CPU%` relativo à capacidade do CT, de 0 a 100 |
| Uma linha por thread | Uma linha por processo |

## Uso

```bash
cttop          # atualiza a cada 1.5s
cttop -d 0.5   # atualiza a cada 0.5s
```

| Tecla | Ação |
|---|---|
| `c` | Ordena os processos por CPU |
| `m` | Ordena os processos por memória |
| `q` | Sai |

O `cttop` só mostra: não mata processo nem muda prioridade.

## O que cada parte mostra

- **CPU:** a barra grande é o consumo do container inteiro, em % da capacidade liberada para ele. Abaixo, o histórico (em terminais com 24 linhas ou mais) e uma barra por núcleo disponível.
- **Linha de resumo:** quantidade de núcleos, limite de CPU se houver, processos rodando agora, espera por CPU (% do tempo nos últimos 10s em que algum processo ficou na fila) e uptime.
- **Memória:** usado em gradiente, cache em azul, e os limites do CT. Swap aparece como "sem swap" quando o CT não tem.
- **Processos:** os que mais consomem, até onde a altura do terminal permitir.

## Como funciona

Tudo vem de `/proc` e `/sys/fs/cgroup`, sem chamar nenhum outro programa:

- Os núcleos visíveis são os da afinidade do processo (`sched_getaffinity`), que dentro do CT é o conjunto de CPUs que o Proxmox liberou.
- O uso total vem do contador de CPU do cgroup do container (`cpu.stat`), então reflete só o CT, não o host.
- Memória e swap vêm de `/proc/meminfo`, que o lxcfs já ajusta para os limites do CT.
- A espera por CPU vem de `cpu.pressure` (PSI).

Funciona com cgroup v2 e v1. Em terminal sem 256 cores usa as cores básicas, e sem UTF-8 desenha em ASCII.

## Limitações

- As barras **por núcleo** vêm de `/proc/stat`. Dependendo da versão do Proxmox e do lxcfs, elas podem refletir o uso daquele núcleo no host inteiro, incluindo outros CTs. A barra de CPU total não tem esse problema.
- Só Linux. Fora de um container também roda, mostrando a máquina inteira.
