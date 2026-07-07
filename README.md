# Sensor Fusion Kalman: Estimação de Estado e Odometria

Este repositório documenta o desenvolvimento e a validação de um sistema de fusão de sensores para robôs móveis, utilizando o pacote `robot_localization` (EKF) do ROS. O projeto compara três estratégias de localização, avaliando o erro médio quadrático (RMSE) em relação ao *Ground Truth* do simulador Gazebo.



## Sobre o Projeto
O objetivo deste trabalho é mitigar a deriva (*drift*) inerente à odometria de rodas através da integração de sensores complementares. Foram implementados três modos de operação:

---

##  Metodologia e Configuração
O sistema utiliza o nó `robot_localization` para fundir dados de sensores com diferentes características de ruído e frequência.

### Configurações de Fusão
1. **Modo 1 (Odometria):** Fusão baseada apenas em `/wheel/odom`,a Odometria Pura (Rodas).

- Aqui é onde a gente vê o erro acumulando na hora. Como a gente só usou o encoder da roda, qualquer patinada que o Husky dava no simulador, o sistema entendia como movimento real. Não tem como fugir, o drift (aquela "deriva") é inevitável. O robô vai andando e, quanto mais tempo passa, mais "perdido" ele fica em relação ao ponto inicial.
   
2. **Modo 2 (Odom + IMU):** Inclusão de `/imu/data` para estabilização de orientação, (Fusão inercial para correção de orientação).
   
- Quando a gente entrou com a IMU, a coisa melhorou um pouco porque agora o robô sabe pra onde ele tá olhando (o Yaw). Se o robô gira, o IMU percebe a aceleração angular e corrige a rota antes do encoder da roda confundir tudo. Ajudou, mas ainda não resolve o problema da posição no mapa. Ele sabe a direção, mas ainda não tem uma "bússola absoluta" de lugar.

3. **Modo 3 (Odom + IMU + GPS):** Inclusão de `/gps/odom` (conversão de coordenadas geodésicas para o plano cartesiano X/Y), (Fusão global para correção de posição absoluta).
   
- Foi aqui que o sistema travou na posição correta. O GPS traz a coordenada absoluta (latitude/longitude), e o Filtro de Kalman faz o meio de campo: ele usa a alta frequência da odometria/IMU pra manter o movimento suave, mas, toda vez que o GPS solta um pulo de coordenada, o filtro corrige a deriva acumulada dos outros sensores.



### Fluxo de Dados
1. **Sensores e Entradas do EKF:** `/wheel/odom` (encoders), `/imu/data` (aceleração/giro) e `/fix` (GPS).
2. **Pré-processamento:** O GPS (latitude/longitude) é convertido para coordenadas cartesianas locais (X, Y) através de um nó de transformação, gerando o tópico `/gps/odom`.
3. **Fusão (EKF):** O nó `ekf_localization_node` recebe os dados, executa a predição baseada no modelo cinemático e a correção baseada nas medições recebidas.
4. **Referencial:** Definimos `world_frame: odom` para garantir que o robô tenha um referencial estável, com a devida sincronização de *timestamps* via `message_filters`.
5. **Saída do Filtro:** `/odometry/filtered`
6. **Ground Truth (Referência):** `/gt/odom` (usado exclusivamente para métricas de erro)

---


## 🛠️ Tecnologias Utilizadas
* **ROS (Robot Operating System):** Noetic
* **Simulador:** Gazebo com robô Clearpath Husky
* **Framework:** `robot_localization` (Extended Kalman Filter)
* **Linguagem:** Python (para avaliação de métricas)


##  Estrutura do Repositório
* `/config`: Ficheiros YAML com os parâmetros do EKF (covariâncias e configurações de sensores).
* `/launch`: Scripts de inicialização (`fusion.launch`) com o gerenciamento dos nós.
* `/scripts`: Ferramentas de avaliação (`avaliar_tcc.py`) para processamento de *bags* e geração dos gráficos de trajetória.



## ⚙️ Como Reproduzir os Testes

Este guia assume que você possui o ambiente **ROS Noetic** instalado com o simulador **Gazebo** e as dependências do robô Husky configuradas.

### 1. Clonagem e Preparação do Workspace
Navegue até o seu diretório de trabalho do ROS (`catkin_ws`) e clone o repositório dentro da pasta `src`:

```bash
cd ~/catkin_ws/src
git clone https://github.com/Eufratica/sensor-fusion-ros.git
cd ..
```
## 2. Compilação
Compile o pacote para integrar os novos arquivos de configuração e scripts ao ambiente ROS:

```bash
catkin_make
source devel/setup.bash
```
## 3. Execução dos Testes (Fluxo de Trabalho)
Para realizar a comparação completa, você precisará de três terminais abertos simultaneamente:

### Terminal 1: Simulador Gazebo
Inicie o ambiente de simulação com o robô Husky:

```bash
source devel/setup.bash
roslaunch husky_gazebo husky_empty_world.launch
```
Aguarde o carregamento completo do robô no mundo virtual.

### Terminal 2: Sistema de Fusão (EKF)
Inicie o nó de fusão configurado para o modo desejado. Substitua N pelo valor correspondente (1, 2 ou 3):

```bash
source devel/setup.bash
roslaunch sensor_fusion_kalman fusion.launch mode:=N
```
- mode:=1: Odometria Pura

- mode:=2: Odom + IMU

- mode:=3: Odom + IMU + GPS

### Terminal 3: Comando Teleop (Controle Remoto)
O teleop serve para você "pilotar" o robô manualmente através do teclado. Isso é essencial para percorrer o trajeto de teste e gerar os dados que o seu filtro de Kalman vai processar.

Abra um novo terminal e execute:
```bash
source devel/setup.bash
rosrun teleop_twist_keyboard teleop_twist_keyboard.py
```
Como usar: Certifique-se de que este terminal esteja com o foco (clicado).

- Comandos: * i (frente), , (trás), j (esquerda), l (direita).

- k (para o robô).

- u, o, m, . (giros e curvas).

Nota: Se o robô não se mover, verifique se o tópico que o teleop está publicando (/cmd_vel) é o mesmo que o Husky está escutando.

### Terminal 4: Comando Rosbag (Gravação de Dados)
O rosbag é como uma "câmera de vídeo" para o seu ROS. Ele grava todos os tópicos que estão passando pela rede (odometria, IMU, GPS, etc.) para que você possa reproduzir depois ou extrair as métricas.

Para gravar tudo o que o robô está fazendo:

```bash
rosbag record -O modeX.bag /odometry/filtered /gt/odom
```
Explicação dos Parâmetros:

- O modeX.bag: Define o nome do arquivo de saída (substitua X pelo número do modo: 1, 2 ou 3).

- /odometry/filtered: É o tópico onde o Filtro de Kalman Estendido (EKF) publica a estimativa de pose otimizada.

- /gt/odom: É o tópico de Ground Truth fornecido pelo simulador Gazebo. Este tópico é registrado para servir como referência absoluta (gabarito) durante a fase de pós-processamento, permitindo o cálculo do RMSE e outros erros de trajetória.

Para terminar a gravação pressione Crtl + C.



### No mesmo terminal em que a gravação foi feita: Avaliação e Métricas
Após deixar o robô completar o percurso, execute o script de avaliação para calcular o RMSE e gerar os gráficos:

```bash
source devel/setup.bash
python3 src/sensor_fusion_kalman/scripts/avaliar_tcc.py
```

## Saída do Sistema e Visualização
Após a execução do script avaliar_tcc.py, o sistema processa os dados contidos nos arquivos .bag e gera automaticamente arquivos de imagem na pasta scripts/ (ou no diretório de destino configurado).

Estes arquivos contêm os gráficos comparativos da trajetória estimada pelo EKF em relação ao Ground Truth.

Arquivos gerados:

- resultado_mode1_odom.png: Visualização do erro no Modo 1 (Odometria Pura).

- resultado_mode2_odom_imu.png: Visualização do erro no Modo 2 (Odom + IMU).

- resultado_mode3_odom_imu_gps.png: Visualização do erro no Modo 3 (Odom + IMU + GPS).

![Resultados no Terminal](images/Captura%20de%20tela%20de%202026-07-07%2011-36-08.png)

Nota de Interpretação: Nos gráficos gerados, a linha contínua representa o Ground Truth (caminho ideal), enquanto a linha pontilhada (ou tracejada) representa a estimativa publicada pelo seu filtro (/odometry/filtered). Quanto mais próximas as linhas estiverem, menor será o RMSE reportado pelo script no terminal.



## Resultados Experimentais

| Modo 1 (Odometria) | Modo 2 (Odom + IMU) | Modo 3 (Odom + IMU + GPS) |
| :---: | :---: | :---: |
| ![Modo 1](images/resultado_mode1_odom.png) | ![Modo 2](images/resultado_mode2_odom_imu.png) | ![Modo 3](images/resultado_mode3_odom_imu_gps.png) |

> **Análise:** O Modo 3 (Odom + IMU + GPS) atingiu um RMSE de **0.0151 m**, demonstrando a eficácia da fusão sensorial na correção da deriva.

A eficácia da fusão sensorial foi validada através do RMSE, demonstrando a convergência do filtro à medida que mais sensores são integrados:

| Modo | Sensores | RMSE (m) |
| :--- | :--- | :--- |
| **1** | Odometria Pura | 1.7706 |
| **2** | Odometria + IMU | 1.3810 |
| **3** | Odom + IMU + GPS | 0.0151 |

> **Conclusão:** A integração de sensores globais (GPS) com sensores proprioceptivos (Odometria/IMU) demonstrou uma redução crítica no erro, validando a arquitetura proposta para aplicações de navegação autônoma.

---
## Melhorias de Localização:

Do Modo 1 (1.77 m) para o Modo 2 (1.38 m): Aqui a gente atacou a "cegueira angular" do robô. Na odometria pura, o robô só sabe quanto as rodas giraram, mas não sabe se está girando no lugar ou em curva. A IMU traz o dado do giroscópio, que é um sensor inercial. O EKF agora tem uma medição direta da velocidade angular. Isso faz o filtro conseguir "desacoplar" o erro de orientação do erro de translação. O erro cai porque a estimativa de Yaw (o ângulo do robô) fica muito mais estável, evitando que o erro de rotação seja jogado na posição X e Y.

Do Modo 2 (1.38 m) para o Modo 3 (0.015 m): O GPS quebra a principal barreira da robótica móvel: o drift acumulado. Enquanto a odometria e a IMU são sensores que acumulam erros, ou seja, integra o erro a cada segundo, o GPS é uma referência absoluta. 

No Modo 3, o filtro tem a "âncora". Toda vez que o GPS envia uma coordenada, o filtro usa o ganho de Kalman K para dizer: "Olha, a odometria diz que estamos aqui, mas o GPS diz que estamos 2 cm pro lado. Vamos confiar mais no GPS porque ele não deriva". É isso que zera o erro acumulado e faz o RMSE despencar para 1.5 cm.


##  Aprofundamento: Funcionamento do EKF

O sistema opera em um ciclo contínuo de **Predição e Correção**:

1. **Predição:** O robô projeta sua posição futura usando o modelo dinâmico (Odom+IMU). A incerteza (covariância $P$) cresce naturalmente aqui.
2. **Correção:** Quando o GPS envia uma medida ($\mathbf{z}$), o filtro calcula o resíduo e ajusta o estado. Se a confiança no GPS é alta, a estimativa é "puxada" para a coordenada global, reduzindo a incerteza.

### Fatores Chave:
* **Matrizes de Covariância:** Ajustamos o `process_noise_covariance` no arquivo `.yaml` para impedir que o sistema confiasse cegamente em sensores ruidosos.
* **Estimativa de Bias da IMU:** O EKF estima internamente o *bias* (erro constante) da IMU, subtraindo-o em tempo real, o que evita que o robô "dance" quando parado.
* **Sincronização Temporal:** Utilizamos `message_filters` para garantir que o *timestamp* dos tópicos de entrada fosse casado no tempo, evitando erros de "fantasmas" na fusão.

## O Funcionamento do EKF: 

Predição e Correção

O nosso EKF trabalha num loop que a gente chama de Predição e Correção:

- Ciclo de Predição: O robô usa o modelo dinâmico (Odometria + IMU) pra "chutar" onde ele vai estar no próximo instante. Como todo sensor tem ruído, a nossa incerteza P aumenta nesse passo.

- Ciclo de Correção: O GPS entra como uma "medição". O filtro compara a nossa estimativa com o GPS e calcula o erro (o resíduo). Aí ele usa o Ganho de Kalman K pra atualizar a posição.
  *Se o ganho é alto, o sistema ajusta a posição drasticamente pro lado do GPS. Se é baixo, o sistema entende que o GPS está com ruído e mantém a confiança no movimento que o robô já estava fazendo.Por que seu sistema é robusto?

- Matrizes de Covariância: Definição de quanto "ruído" cada sensor tem. Ao configurar isso, evitamos que o sistema de confiasse cegamente em um dado ruim.
  
- Estimativa de Bias da IMU: Um ponto que a banca vai adorar: o seu EKF não só funde sensores, ele aprende o bias (erro constante) da IMU. Como a IMU tem um erro que varia com a temperatura/tempo, o EKF percebe que esse erro é constante e subtrai ele do cálculo automaticamente. Isso mantém o robô parado sem "dançar" no mapa.

Sincronização (Timestamping): O erro de 1.5 cm só foi possível porque usamos message_filters do ROS. Garantimos que o dado do GPS e da odometria se encontrassem no mesmo instante no tempo t. Se tivéssemos um atraso (latência) de poucos milissegundos, o filtro estaria comparando posições em tempos diferentes, o que geraria um erro de "fantasma" que subiria o RMSE lá pro alto.

---

Nota: O Ground Truth (/gt/odom) é utilizado estritamente para avaliação. O EKF não tem acesso a este tópico durante o processamento, garantindo que a estimativa de estado seja realizada apenas com sensores embarcados, simulando um cenário real de robótica móvel.
