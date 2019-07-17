#centos6のイメージを取得
FROM centos:centos6

#タイムゾーンの設定
RUN /bin/cp /usr/share/zoneinfo/Asia/Tokyo /etc/localtime

#yumリポジトリの追加
RUN yum install -y epel-release
RUN rpm -Uvh http://rpms.famillecollet.com/enterprise/remi-release-6.rpm

#php7のインストール
RUN yum install -y --enablerepo=remi,remi-php70 php php-devel php-mbstring php-pdo php-mysql php-gd php-mcrypt libmcrypt

#yumによるHTTPD,MySQL,tar,wgetのインストール
RUN yum -y install httpd mysql-server tar wget

#tmpディレクトリに移動
WORKDIR /tmp/

#wordpress一式のダウンロード
RUN wget https://ja.wordpress.org/latest-ja.tar.gz

#wordpressの展開
RUN tar xvfz ./latest-ja.tar.gz -C /var/www/html

#ダウンロードしたwordpressの削除
RUN rm -f ./latest-ja.tar.gz

#wordpressのconfigファイルをリネームして利用可能にする
RUN cp /var/www/html/wordpress/wp-config-sample.php /var/www/html/wordpress/wp-config.php

#wordpressのconfigファイルに必要な情報をsedコマンドで書き換える
RUN sed -i -e 's/database_name_here/wordpress/g' -e 's/username_here/wordpress/g' -e 's/password_here/wppass/g' /var/www/html/wordpress/wp-config.php

#DocumentRootディレクトリの所有者をapacheに変更
RUN chown -R apache.apache /var/www/html/

#mysqldの起動、DB作成、ユーザ作成および権限設定、mysqldの停止
RUN service mysqld start && mysql -u root -e "CREATE DATABASE wordpress; GRANT ALL PRIVILEGES ON wordpress.* TO 'wordpress'@'localhost' IDENTIFIED BY 'wppass'; FLUSH PRIVILEGES;" && service mysqld stop

#mysqld,httpdの起動スクリプトの作成
RUN echo -e "service mysqld start\nservice httpd start\n/bin/bash" > /startService.sh

#mysqld,httpdの起動スクリプトの権限設定
RUN chmod o+x /startService.sh

#公開ポート
EXPOSE 80

#mysqld,httpdの起動スクリプトの実行
CMD /startService.sh
