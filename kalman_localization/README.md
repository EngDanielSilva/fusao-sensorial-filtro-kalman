# kalman_localization

Pacote ROS (Noetic) para comparar 3 configurações de localização do Husky
no ambiente `lar_gazebo`, usando `robot_localization` (EKF):

1. **odom** — apenas odometria de rodas
2. **odom_imu** — odometria + IMU
3. **odom_imu_gps** — odometria + IMU + GPS

O GPS é simulado via `hector_gazebo_plugins` (`/fix`, `sensor_msgs/NavSatFix`)
e convertido para `nav_msgs/Odometry` em `/gps/odom` pelo nó
`scripts/gps_to_odom.py` (projeção equirretangular local). O ground truth
(`/gt/odom`) vem de um plugin P3D e é usado **só para avaliação**, nunca como
entrada do filtro.

## 1. Instalação

Clone este pacote **dentro do mesmo workspace** do `lar_gazebo` (ele depende
do `lar_gazebo` já estar buildado), por exemplo:

```bash
~/catkin_ws/src/
├── lar_gazebo/            # repositório do curso
└── kalman_localization/   # este pacote
```

### Usando o Docker do `lar_gazebo`

Dentro do repositório `lar_gazebo`, crie `docker-compose.override.yml`
montando este pacote como um segundo volume:

```yaml
services:
  lar_gazebo:
    volumes:
      - /caminho/absoluto/para/kalman_localization:/ws/src/kalman_localization:rw
```

Depois:

```bash
./scripts/shell.sh
# dentro do container:
cd /ws
catkin_make   # ou catkin build
source devel/setup.bash
```

### Instalação nativa

```bash
sudo apt install ros-noetic-robot-localization ros-noetic-hector-gazebo-plugins
cd ~/catkin_ws && catkin_make
source devel/setup.bash
```

## 2. Como executar um teste

Terminal 1 — sobe o mundo + Husky + GPS/ground truth:

```bash
roslaunch kalman_localization husky_kalman.launch
```

Terminal 2 — roda o experimento completo (EKF + trajetória automática +
gravação do bag + cálculo de métricas) para uma das 3 configurações:

```bash
rosrun kalman_localization run_experiment.sh odom
rosrun kalman_localization run_experiment.sh odom_imu
rosrun kalman_localization run_experiment.sh odom_imu_gps
```

Repita o terminal 1 (reiniciar a simulação) entre cada execução do terminal 2,
para que as 3 trajetórias comecem do mesmo estado inicial.

Resultados de cada configuração ficam em:

```
results/<config>/metrics.csv
results/<config>/summary.txt
results/<config>/trajetoria.png
results/<config>/erro_posicao.png
results/<config>/erro_orientacao.png
```

## 3. Rodando manualmente (sem o script)

```bash
roslaunch kalman_localization ekf_odom.launch          # ou ekf_odom_imu / ekf_odom_imu_gps
rosrun kalman_localization drive_pattern.py
rosbag record -O results/odom.bag /odometry/filtered /gt/odom
rosrun kalman_localization compute_metrics.py results/odom.bag odom results
```

## 4. Estrutura

```
kalman_localization/
├── urdf/kalman_extras.urdf       # plugins P3D (gt) + GPS (hector)
├── launch/husky_kalman.launch    # husky + sensores + gps_to_odom
├── launch/ekf_odom.launch        # EKF: odom
├── launch/ekf_odom_imu.launch    # EKF: odom + imu
├── launch/ekf_odom_imu_gps.launch# EKF: odom + imu + gps
├── config/ekf_*.yaml             # parâmetros de cada EKF
├── scripts/gps_to_odom.py        # NavSatFix -> /gps/odom
├── scripts/drive_pattern.py      # trajetória repetível (quadrado)
├── scripts/run_experiment.sh     # orquestra um teste completo
├── scripts/compute_metrics.py    # RMSE, erro final, erro de orientação
└── results/                      # bags, csv e gráficos gerados
```

## 5. Discussão dos resultados (preencher após executar)

> Depois de rodar as 3 configurações, complete esta seção com:
> - Tabela comparando RMSE de posição, erro final e RMSE de orientação das 3 configs.
> - Qual configuração teve melhor desempenho e por quê (esperado: odom_imu_gps,
>   pois o GPS corrige o drift acumulado da odometria/IMU).
> - Limitações: GPS simulado com referência fixa em (0,0) e projeção local
>   simplificada (válida apenas para áreas pequenas); IMU e odometria do
>   Husky em Gazebo têm ruído idealizado em relação ao hardware real.

| Configuração   | RMSE posição (m) | Erro final (m) | RMSE orientação (rad) |
|----------------|-------------------|-----------------|-------------------------|
| odom           |                   |                 |                         |
| odom_imu       |                   |                 |                         |
| odom_imu_gps   |                   |                 |                         |
