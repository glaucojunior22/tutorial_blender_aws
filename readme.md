# Blender render na AWS
## Tutorial com passo a passo para renderizar projetos do blender utilizando instâncias EC2 da AWS

A Amazon possui instâncias próprias para tarefas que necessitem de uso intensivo de GPU (placa de vídeo), atualmente (Abril/2018) a versão mais potente disponível são as instâncias P3 com a GPU NVIDIA Tesla V100. Geralmente essas instâncias não estão disponíveis, sendo necessário solicitar um aumento de limite (Menu esquerdo no EC2 Dashboard -> Limits), esse processo deve levar uns 2 dias.

Depois de solicitar o aumento do limite e poder usar esse tipo de instância vamos iniciar as configurações:

1. Iniciar uma nova instância clicando em "Launch Instance"
2. Escolher a AMI (basicamente é o sistema operacional e alguns softwares pré-instalados), existe uma imagem da própria NVIDIA com o Amazon Linux e os drivers Tesla chamado: Amazon Linux AMI with NVIDIA TESLA GPU Driver
3. Escolher o tipo da instância, nesse caso usei a p3.2xlarge
4. Na parte de configurações a única alteração que fiz foi trocar o starage para SSD
5. Ao criar a primeira instância será criada uma chave criptográfica para utilizar o acesso vai SSH, salve esse arquivo .pem e altere as permissões do arquivo com o comando: `$chmod 400 arquivo.pem` pois será necessário para acessar a instância
6. Depois de iniciar a instância acesse via ssh com o comando `$ssh -i arquivo.pem ec2-user@DNS-DA-INSTANCIA`
7. Atualizar os pacotes do sistema operacional: `$sudo yum update`
8. Configurar o clock da placa de vídeo para utilizar o máximo: `$sudo nvidia-smi -ac 877,1530` OBS: Esses valores são para a GPU Tesla V100, para outros modelos verifique os valores aqui: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/optimize_gpu.html
9. Instalar Blender:
  * `$sudo yum -y install freetype freetype-devel libpng-devel`
  * `$sudo yum -y install mesa-libGLU-devel`
  * `$wget (https://builder.blender.org/download/blender-2.79-dda7e3b6956-linux-glibc219-x86_64.tar.bz2)`
  * `$tar -jxvf blender-2.79-dda7e3b6956-linux-glibc219-x86_64.tar.bz2`
  * `$mv blender-2.79-dda7e3b6956-linux-glibc219-x86_64 blender`
  * `$sudo vi ~/.bashrc` e adicionar a ultima linha:
  * `$export PATH=/home/ec2-user/blender:$PATH`
10. Para testar a renderização do Blender podem ser usados os projetos de teste no site blender.org:
  * `$wget https://download.blender.org/demo/test/benchmark.zip`
  * `$unzip benchmark.zip`
11. Aparentemente mesmo que o projeto esteja configurado para renderizar via GPU ele ainda renderiza via CPU, para contornar isso encontrei esse script em Python para forçar a renderização via GPU:
    ```python
    import bpy

    # https://blender.stackexchange.com/questions/5281/blender-sets-compute-device-cuda-but-doesnt-use-it-for-actual-render-on-ec2
    bpy.context.user_preferences.addons['cycles'].preferences.compute_device_type = 'CUDA'
    bpy.context.user_preferences.addons['cycles'].preferences.devices[0].use = True

    bpy.context.scene.cycles.device = 'GPU'

    # A linha abaixo está comentada pois o nome da cena pode ser diferente e dar erro mas caso saiba o nome da cena pode editar e descomentar, isso vai fazer o render gerar o arquivo no caminho abaixo
    # bpy.data.scenes["Scene"].render.filepath = "/tmp/output.png"
    bpy.ops.render.render(write_still=True)
    ```
12. Comando para renderizar o projeto:
    `$blender -b "benchmark/benchmark.blend" -x 1 -noaudio -nojoystick -E CYCLES -t 0 -P "config.py" -y`
13. Para copiar o arquivo gerado para o computador local pode ser usado o scp:
    `$scp -i arquivo.pem ec2-user@DNS-DA_INSTANCIA:benchmark/render/previz_###.png arquivo.png`

A maior parte das informações para esse tutorial foram tiradas de: (https://medium.com/@Mikulas/blender-rendering-on-aws-p2-61f7fb8405d8)
