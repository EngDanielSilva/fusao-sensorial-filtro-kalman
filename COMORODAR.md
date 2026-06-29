# Como rodar o projeto do zero

Este guia assume que você está partindo de uma máquina limpa (com Docker e
Docker Compose instalados) e quer reproduzir os experimentos deste
repositório.

## 0. Pré-requisitos

- Docker e Docker Compose instalados
- Suporte a GUI/X11 se quiser visualizar o Gazebo com interface gráfica
  (opcional — os experimentos rodam sem GUI também)

## 1. Clonar os dois repositórios

O ambiente de simulação (`lar_gazebo`) é fornecido pelo professor/curso e é
**separado** deste pacote. Você precisa dos dois, organizados assim:

```bash
mkdir -p ~/lar_ws
cd ~/lar_ws

# Workspace de simulação (fornecido pelo professor)
git clone https://github.com/lar-deeufba/lar_gazebo.git

# Este pacote (localização com EKF)
git clone https://github.com/EngDanielSilva/fusao-sensorial-filtro-kalman.git
```

Ao final deste passo, a estrutura deve ser:

```
~/lar_ws/
├── lar_gazebo/            (clonado do professor)
└── kalman_localization/   (este repositório)
```

## 2. Conectar o pacote ao workspace via Docker volume

O `lar_gazebo` sobe um container com um workspace catkin em `/ws`. Para que
o container "veja" o pacote `kalman_localization` e para que o build do
catkin persista entre reinicializações do container, crie o arquivo
`~/lar_ws/lar_gazebo/docker-compose.override.yml` com o seguinte conteúdo
(substitua `<SEU_USUARIO>` pelo seu usuário/home real):

```yaml
services:
  lar_gazebo:
    volumes:
      - /home/<SEU_USUARIO>/lar_ws/kalman_localization:/ws/src/kalman_localization:rw
      - /home/<SEU_USUARIO>/lar_ws/_build:/ws/build:rw
      - /home/<SEU_USUARIO>/lar_ws/_devel:/ws/devel:rw
```

> Esse arquivo é específico da sua máquina (tem paths absolutos), por isso
> não faz parte deste repositório — você cria ele localmente seguindo este
> guia. Sem ele, o pacote não aparece dentro do container e o build não
> persiste entre `docker compose down`/`up`.

## 3. Subir o ambiente (uma vez por sessão de trabalho)

```bash
cd ~/lar_ws/lar_gazebo
docker compose up -d
./scripts/shell.sh
```

Isso te coloca dentro de um container novo
(`lar_gazebo-lar_gazebo-run-<hash>`). Em **outro** terminal do host, rode
`docker ps` e anote o nome real do container — você vai precisar dele para
abrir os próximos terminais com `docker exec`.

## 4. Build (apenas na primeira vez, ou se apagar `_build`/`_devel`)

Dentro do container:

```bash
cd /ws
catkin build
source devel/setup.bash
```

Por causa dos volumes configurados no passo 2, o build persiste entre
recriações do container — só repita este passo se apagar as pastas
`~/lar_ws/_build` / `~/lar_ws/_devel`, ou se trocar de máquina.

## 5. Variáveis de ambiente obrigatórias antes de cada `roslaunch`

Estas variáveis **não persistem** entre terminais novos — exporte de novo em
**todo** terminal que vai rodar o `roslaunch` principal, mesmo dentro do
mesmo container:

```bash
export ENABLE_EKF=false
export HUSKY_URDF_EXTRAS=$(rospack find kalman_localization)/urdf/kalman_extras.urdf
```

- `ENABLE_EKF=false` desliga o EKF padrão do Husky (`husky_control`), que
  senão conflita de nome com os EKFs deste pacote.
- `HUSKY_URDF_EXTRAS` garante que o ground truth seja publicado em
  `/gt/odom` (plugin P3D, definido em `kalman_extras.urdf`). Se essa
  variável não for setada corretamente, o tópico de ground truth não existe
  no nome esperado e as métricas falham.

## 6. Terminal 1 — subir a simulação

```bash
docker exec -it <nome_do_container> bash
source /ws/devel/setup.bash
export ENABLE_EKF=false
export HUSKY_URDF_EXTRAS=$(rospack find kalman_localization)/urdf/kalman_extras.urdf
roslaunch kalman_localization husky_kalman.launch
```

No SUMMARY impresso ao subir, confira que **não** aparece nenhum nó
`ekf_*` na lista de NODES — esse launch só sobe mundo + Husky + sensores.
Se aparecer `ekf_localization`, o `ENABLE_EKF` não foi aplicado
corretamente.

Deixe este terminal rodando.

## 7. Terminal 2 — rodar um experimento

Em outro terminal, no **mesmo** container:

```bash
docker exec -it <nome_do_container> bash
source /ws/devel/setup.bash
export ENABLE_EKF=false
rosrun kalman_localization run_experiment.sh odom
```

Repita depois para `odom_imu` e `odom_imu_gps`.

**Entre cada execução**, reinicie a simulação para o robô voltar à posição
inicial:

1. No Terminal 1: `Ctrl+C`
2. Confirme que não sobrou processo zumbi:
   ```bash
   ps aux | grep -E "gzserver|gzclient|rosmaster|roscore|roslaunch"
   ```
   Se sobrar algo, `kill -9 <PIDs>` e cheque de novo.
3. Repita o bloco do passo 6 (incluindo os `export`).

## 8. Verificar os resultados

Ao terminar, `run_experiment.sh` já imprime as métricas na tela e salva os
artefatos em `results/<config>/`. Se aparecer o erro:

```
ERRO: nao encontrei dados em /odometry/filtered e/ou /gt/odom no bag.
```

o bag não tem dados válidos. Diagnóstico rápido:

```bash
rosbag info /ws/src/kalman_localization/results/<config>.bag
```

Um bag válido tem cerca de 80–150 s de duração, ~2 mil mensagens, e os dois
tópicos `/gt/odom` e `/odometry/filtered` listados.

## 9. Resultados esperados (referência)

| Configuração   | RMSE pos (m) | Erro final pos (m) | RMSE orient (rad) | Erro final orient (rad) |
|-----------------|:------------:|:-------------------:|:------------------:|:-------------------------:|
| odom            | 9.5438       | 12.2382             | 1.7556              | 2.4869                     |
| odom_imu        | 5.7791       | 5.4969              | 0.0024              | 0.0000                     |
| odom_imu_gps    | 5.4338       | 6.1530              | 0.0029              | 0.0000                     |

Para a discussão desses resultados, veja o `README.md` principal.
