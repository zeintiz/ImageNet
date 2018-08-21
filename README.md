<h1>Trial on kaggle imagenet object localization by yolov3 in google cloud</h1>
project:https://www.kaggle.com/c/imagenet-object-localization-challenge<br>
yolo:https://pjreddie.com/darknet/yolo/<br>
nice yolo explanation:https://medium.com/@jonathan_hui/real-time-object-detection-with-yolo-yolov2-28b1b93e2088<br>
My operation system is Mac OS, although it doesn't matter much, because we do most steps on google cloud.<br>
Let's do it step by step:<br>
<h2>1.basic environment preparation:</h2>
<h4>I.Apply a google cloud account.</h4>
ps:Google provide $300 credit trial time for first time sign up account.
<h4>II.Create a ubuntu 16.04 compute engine instance on google cloud, with 450G SSD disk, 4 cores' cpu and, 15G memories. </h4>
we will change it a little bit later coz GPU/CPU extention or other reasons, but now it's enough.<br>
<h4>III.Apply quotas increasing on Nvidia tesla K80/P100/V100, coz we don't have permission to use gpu default. </h4>
GPUs cost credit so fast, so we can choose it by needed. For me, I just increase 1 K80s for test, 4 P100s for training our model, haven't tried on V100 yet.<br>
<h4>IV.SSH connection by RSA keys.</h4>
#:ssh-keygen -t rsa -f ~/.ssh/gc_rsa -C anynamehere<br>
No pass word is easy for login.<br>
#:cd ~/.ssh<br>
#:vi gc_rsa.pub<br>
then go to google cloud, copy everything in gc_rsa.pub to ubuntu instance SSH key part.<br>
#:chmod 400 gc_rsa<br>
#:ssh -i gc_rsa anynamehere@your google cloud external ip<br>
we can also connect by 'FileZilla', no more words here.<br>
<h4>V.pip installation</h4>
#:sudo apt update<br>
#:sudo apt upgrade<br>
#:sudo apt-get -y install python-pip<br>
#:sudo apt-get -y install python3-pip<br>
<h4>VI.kaggle-cli installation</h4>
#:pip install kaggle-cli<br>
<h2>2.dataset download</h2>
#:kg download -u &lt;your kaggle username&gt; -p &lt;your kaggle password&gt; -c imagenet-object-localization-challenge<br>
// dataset is about 160G, so it will cost about 1 hour if your instance download speed is around 42.9 MiB/s.<br>
// let's open another ssh connection to do next step.<br>
<h2>3.opencv-3.4.0 installation(we will turn on opencv option in yolo project later for better image processing)</h2>
execute all the steps in the following url.<br>
http://www.python36.com/how-to-install-opencv340-on-ubuntu1604/<br>
<h2>4.cuda 9.0 with cudnn 7.0 installation</h2>
we can use the fallowing bash script, download it and execute it in instance.<br>
https://gist.github.com/ashokpant/5c4e9481615f54af4025ab2085f85869#file-cuda_9-0_cudnn_7-0-sh<br>
<h2>5.cudnn library configuration</h2>
go to https://developer.nvidia.com/rdp/cudnn-download to download cuDNN v7.0.5 Library for Linux CUDA 9.0<br>
it's name should be cudnn-9.0-linux-x64-v7.tgz, we use scp command or filezilla to move this package from local machine to remote instance.<br>
#:scp -i ~/.ssh/gc_rsa Downloads/cudnn-9.0-linux-x64-v7.tgz anynamehere@your google cloud external ip:~/<br>
// come to instance window<br>
#:tar zxvf cudnn-9.0-linux-x64-v7.tgz<br>
#:cd cuda<br>
#:sudo cp include/* /usr/local/cuda-9.0/include/<br>
#:sudo cp lib64/* /usr/local/cuda-9.0/lib64/<br>
#:echo 'export PATH=/usr/local/cuda-9.0/bin:$PATH' >> ~/.bashrc<br>
#:echo 'export LD_LIBRARY_PATH=/usr/local/cuda-9.0/lib64/:$LD_LIBRARY_PATH' >> ~/.bashrc<br>
#:source ~/.bashrc<br>
<h2>6.Add 1 piece of K80 GPU we applied before on our instance when it's power off, then start it.</h2>
Maybe the external ip changed after restart, but way to connect it is same as before.<br>
<h2>7.Yolo installation</h2>
#:git clone https://github.com/pjreddie/darknet<br>
#:cd darknet<br>
#:make<br>
<h2>8.X11 installatoin both of instance and our local machine, so that we can see our predicted image remotely.</h2>
#:sudo apt-get install xorg openbox<br>
// what I need on my mac is XQuartz.<br>
// install feh, so that we can see any picture remotely.<br>
#:sudo apt install feh<br>
// logout from instance, connect it with additional parameter, then test it<br>
#ssh -Y -i ~/.ssh/gc_rsa anynamehere@your google cloud external ip
#:feh darknet/data/dog.jpg<br>
<h2>9.test yolov3</h2>
// Actually we've done a good job until now, but we still can't see expected result if we won't change Makefile a little bit,<br>
// I haven't figured out the reason, although let's just change it now. <br>
#:cd darknet
#:sed -i 's/CUDNN=1/CUDNN=0/g' Makefile<br>
#:make<br>
#:wget https://pjreddie.com/media/files/yolov3.weights<br>
#:./darknet detector test cfg/coco.data cfg/yolov3.cfg yolov3.weights data/dog.jpg<br>
<h2>10.Now let's come to the main part - train yolo on kaggle imagenet object localization</h2>
<h4>I.training data preprocessing.</h4>
#:cd ~<br>
#:tar zxvf imagenet_object_localization.tar.gz<br>
#:unzip LOC_synset_mapping.txt.zip<br>
#:mkdir ILSVRC/Data/CLS-LOC/train/images<br>
#:mv ILSVRC/Data/CLS-LOC/train/n* ILSVRC/Data/CLS-LOC/train/images/<br>
#:mv ILSVRC/Data/CLS-LOC/val/ ILSVRC/Data/CLS-LOC/images<br>
#:mkdir ILSVRC/Data/CLS-LOC/val/<br>
#:mv ILSVRC/Data/CLS-LOC/images ILSVRC/Data/CLS-LOC/val/images<br>
#:git pull https://github.com/mingweihe/ImageNet<br>
#:pip3 install pandas<br>
#:pip3 install pathlib<br>
#:cd ImageNet<br>
#:python3 generate_labels.py ../LOC_synset_mapping.txt ../ILSVRC/Annotations/CLS-LOC/train ../ILSVRC/Data/CLS-LOC/train/labels 1<br>
#:python3 generate_labels.py ../LOC_synset_mapping.txt ../ILSVRC/Annotations/CLS-LOC/val ../ILSVRC/Data/CLS-LOC/val/labels 0<br>
#:cd ~<br>
#:find `pwd`/ILSVRC/Data/CLS-LOC/train/labels/ -name \*.txt > darknet/data/inet.train.list<br>
#:sed -i 's/\.txt/\.JPEG/g' darknet/data/inet.train.list<br>
#:sed -i 's/labels/images/g' darknet/data/inet.train.list<br>

#:find `pwd`/ILSVRC/Data/CLS-LOC/val/labels/ -name \*.txt > darknet/data/inet.val.list<br>
#:sed -i 's/\.txt/\.JPEG/g' darknet/data/inet.val.list<br>
#:sed -i 's/labels/images/g' darknet/data/inet.val.list<br>
<h4>II.pretrained weights preparation.</h4>
#:cd darknet<br>
#:wget https://pjreddie.com/media/files/darknet53.conv.74<br>
<h4>III.cfg files preparation</h4>
#:cp ~/ImageNet/yolov3-ILSVRC.cfg cfg/<br>
#:cp ~/ImageNet/ILSVRC.data cfg/<br>
<h4>IV.Traning</h4>
#:./darknet detector train cfg/ILSVRC.data cfg/yolov3-ILSVRC.cfg darknet53.conv.74<br>
// we can also restart training from a checkpoint:<br>
#:./darknet detector train cfg/ILSVRC.data cfg/yolov3-ILSVRC.cfg backup/yolov3.backup<br>
<h4>V.Traininguse with multiple GPUs</h4>
// shutdown instance, configure GPU from 1 piece of K80 to 4 piece of P100, with 8 CPUs.<br>
// boot instance, start training using following command<br>
#:./darknet detector train cfg/ILSVRC.data cfg/yolov3-ILSVRC.cfg darknet53.conv.74 -gpus 0,1,2,3<br>
// continue from checkpoints we can replace darknet53.conv.74 with backup file.<br>
<h4>VI.</h4>
<h4>VII.</h4>
<h4>VIII.</h4>
<h2>11.Prediction</h2>
#:<br>
<h2>12.transfer predcitions to CSV file.</h2>
<h2>13.submit our predictions.</h2>
Good luck and thanks for your attention.<br>













