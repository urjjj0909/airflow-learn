# airflow-learn
本文主要參照[一段Airflow與資料工程的故事：談如何用Python追漫畫連載](https://leemeng.tw/a-story-about-airflow-and-data-engineering-using-how-to-use-python-to-catch-up-with-latest-comics-as-an-example.html#app-v2)作為Airflow的入門，由於這篇文章已經講解得非常好，僅在此記錄實作過程中遇到的一些問題和解法。

## Windows下安裝Linux環境、Anaconda
到Microsoft Store當中搜尋Ubuntu 18.04 LTS版本，安裝後即可使用Linux terminal，開啟後就可以執行以下程式碼把整個專案複製下來：

    git clone https://github.com/leemengtaiwan/airflow-tutorials.git
    cd airflow-tutorials

由於Airflow是利用Python做操作，在這裡直接先透過以下指令下載Anaconda環境：

    wget https://repo.anaconda.com/archive/Anaconda3-2020.07-Linux-x86_64.sh
    
接著再執行安裝：

    bash Anaconda3-2020.07-Linux-x86_64.sh
    
安裝過程中需先閱讀完授權條款、更改安裝路徑、是否要執行conda init的設定等等，可以參考[Office指南](https://officeguide.cc/ubuntu-linux-install-anaconda-data-science-platform-tutorial)講解，然而未來在terminal中呼叫conda是否會直接載入base環境就單純依個人習慣而定。

在安裝完後須先執行以下指令，<username>則是填入安裝完Ubuntu時設定的使用者名稱：

    export PATH=/home/<username>/anaconda3/bin:$PATH
    source ~/.bashrc

基本上到這裡就可以呼叫conda檢查是否有正常執行，然而比較好的方式是寫進`~/.bashrc`裡面，這樣以後開啟terminal就會自動幫你做好source並且正常使用Anaconda，以上程式碼就可以改寫成「把`export PATH=/home/<username>/anaconda3/bin:$PATH`寫入`~/.bashrc`中」：

    echo 'export PATH=/home/<username>/anaconda3/bin:$PATH' >> ~/.bashrc

當然我們也可以直接用`nano ~/.bashrc`開啟檔案後自行新增程式碼進去。

## 建立Airflow環境
依照教學中的指示執行以下程式碼，需要注意的一點是初始化Airflow Metadata DB的寫法目前版本有稍做更動：
  
    conda create -n airflow-tutorials python=3.6 -y
    source activate airflow-tutorials
    pip install "apache-airflow[crypto, slack]"
    export AIRFLOW_HOME="$(pwd)"
    airflow db init

在上述程式碼中，有二點要特別注意：
  
* 如同上一小節所述，比較好的方式也是把`AIRFLOW_HOME`寫入`~/.bashrc`中，也就是`echo 'export AIRFLOW_HOME="$(pwd)"' >> ~/.bashrc`
  
* 若沒有設定`export AIRFLOW_HOME="$(pwd)"`就做`airflow db init`的話，那麼Airflow就會使用當初作者測試時使用的路徑，裡面會有很多的example可以供參考(也是很棒的教學參考)，但就不會有你git clone下來的repo的路徑而發生找不到[一段Airflow與資料工程的故事：談如何用Python追漫畫連載](https://leemeng.tw/a-story-about-airflow-and-data-engineering-using-how-to-use-python-to-catch-up-with-latest-comics-as-an-example.html#app-v2)中的教學檔案

此外，在Airflow環境第一次建立時會需要username或password等等的設定，執行以下程式碼並填入括弧內資訊即可，未來就會作為登入Airflow環境的帳號密碼：
  
    airflow users create --role <role> --username <username> --email <email> --firstname <firstname> --lastname <lastname> --password <password>

都完成後就可以執行以下程式碼看到Airflow Web UI的介面啦！

    airflow webserver -p 8080
