# Sensor Fusion Kalman: Estimação de Estado e Odometria

Este repositório documenta o desenvolvimento e a validação de um sistema de fusão de sensores para robôs móveis, utilizando o pacote `robot_localization` (EKF) do ROS. O projeto compara três estratégias de localização, avaliando o erro médio quadrático (RMSE) em relação ao *Ground Truth* do simulador Gazebo.



## 🚀 Sobre o Projeto
O objetivo deste trabalho é mitigar a deriva (*drift*) inerente à odometria de rodas através da integração de sensores complementares. Foram implementados três modos de operação:

---

## 📋 Metodologia e Configuração
O sistema utiliza o nó `robot_localization` para fundir dados de sensores com diferentes características de ruído e frequência.

### Configurações de Fusão
1. **Modo 1 (Odometria):** Fusão baseada apenas em `/wheel/odom`.
2. **Modo 2 (Odom + IMU):** Inclusão de `/imu/data` para estabilização de orientação.
3. **Modo 3 (Odom + IMU + GPS):** Inclusão de `/gps/odom` (conversão de coordenadas geodésicas para o plano cartesiano X/Y).

### Tópicos Utilizados
* **Entradas do EKF:** `/wheel/odom`, `/imu/data`, `/gps/odom`
* **Saída do Filtro:** `/odometry/filtered`
* **Ground Truth (Referência):** `/gt/odom` (usado exclusivamente para métricas de erro)

---

1. **Modo 1:** Odometria Pura (Rodas).
2. **Modo 2:** Odometria + IMU (Fusão inercial para correção de orientação).
3. **Modo 3:** Odom + IMU + GPS (Fusão global para correção de posição absoluta).

## 🛠️ Tecnologias Utilizadas
* **ROS (Robot Operating System):** Noetic
* **Simulador:** Gazebo com robô Clearpath Husky
* **Framework:** `robot_localization` (Extended Kalman Filter)
* **Linguagem:** Python (para avaliação de métricas)

## 📊 Resultados Obtidos
A eficácia da fusão sensorial foi validada através do RMSE, demonstrando a convergência do filtro à medida que mais sensores são integrados:

| Modo | Sensores | RMSE (m) |
| :--- | :--- | :--- |
| **1** | Odometria Pura | 1.7706 |
| **2** | Odometria + IMU | 1.3810 |
| **3** | Odom + IMU + GPS | 0.0151 |

> **Conclusão:** A integração de sensores globais (GPS) com sensores proprioceptivos (Odometria/IMU) demonstrou uma redução crítica no erro, validando a arquitetura proposta para aplicações de navegação autônoma.

## 📂 Estrutura do Repositório
* `/config`: Ficheiros YAML com os parâmetros do EKF (covariâncias e configurações de sensores).
* `/launch`: Scripts de inicialização (`fusion.launch`) com o gerenciamento dos nós.
* `/scripts`: Ferramentas de avaliação (`avaliar_tcc.py`) para processamento de *bags* e geração dos gráficos de trajetória.



## ⚙️ Como Reproduzir
1. Certifique-se de ter o ambiente ROS Noetic instalado.
2. Clone o repositório:
   ```bash
   git clone https://github.com/Eufratica/sensor-fusion-ros.git

## 📊 Resultados Experimentais

| Modo 1 (Odometria) | Modo 2 (Odom + IMU) | Modo 3 (Odom + IMU + GPS) |
| :---: | :---: | :---: |
| ![Modo 1](images/resultado_mode1_odom.png) | ![Modo 2](images/resultado_mode2_odom_imu.png) | ![Modo 3](images/resultado_mode3_odom_imu_gps.png) |

> **Análise:** O Modo 3 (Odom + IMU + GPS) atingiu um RMSE de **0.0151 m**, demonstrando a eficácia da fusão sensorial na correção da deriva.
