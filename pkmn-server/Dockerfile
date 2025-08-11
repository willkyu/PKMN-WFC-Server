###
### Dockerfile migration of https://github.com/EnergyCube/cowfc_installer
###
### `docker-compose up`
###
### Open ports listed below
### TCP: 53, 80, 443, 8000, 9000, 9001, 9009, 9002, 9003, 9998, 27500, 27900, 27901, 28910, 29900, 29901, 29920
### UDP: 53
###
### Admin URL: http://$IP/?page=admin&section=Dashboard
###

ARG ADMIN_USERNAME
ARG ADMIN_PASSWORD

ARG VERSION_OPENSSL="openssl-1.1.1m"
ARG VERSION_HTTPD="httpd-2.4.52"

# =========================
# Build OpenSSL (with SSLv3)
# =========================
FROM debian:11 AS builder_openssl
ARG VERSION_OPENSSL
ARG VERSION_HTTPD
ENV DEBIAN_FRONTEND=noninteractive
RUN set -eux; \
    apt update && apt -y install \
        build-essential curl libapr1-dev libaprutil1-dev libpcre3-dev; \
    cd /; \
    OPENSSL_TARBALL="${VERSION_OPENSSL}.tar.gz"; \
    curl -fL "https://www.openssl.org/source/${OPENSSL_TARBALL}" -o "${OPENSSL_TARBALL}" \
      || curl -fL "https://www.openssl.org/source/old/1.1.1/${OPENSSL_TARBALL}" -o "${OPENSSL_TARBALL}"; \
    tar xvf "${OPENSSL_TARBALL}"; rm -f "${OPENSSL_TARBALL}"; \
    cd "${VERSION_OPENSSL}"; \
    ./config --prefix=/usr/local/openssl --openssldir=/usr/local/openssl \
      shared enable-ssl3 enable-ssl3-method enable-weak-ssl-ciphers; \
    make install_sw; \
    make install_ssldirs; \
    cd /; \
    curl -fL "http://archive.apache.org/dist/httpd/${VERSION_HTTPD}.tar.bz2" -o "${VERSION_HTTPD}.tar.bz2"; \
    tar xvf "${VERSION_HTTPD}.tar.bz2"; rm "${VERSION_HTTPD}.tar.bz2"; \
    cd "${VERSION_HTTPD}"; \
    echo "/usr/local/openssl/lib" > /etc/ld.so.conf.d/usr.local.openssl.lib.conf && ldconfig; \
    ./configure --enable-ssl --with-ssl=/usr/local/openssl/ && make

# =========================
# Dummy certs
# =========================
FROM debian:11 AS builder_dummy-certs
ENV DEBIAN_FRONTEND=noninteractive
RUN set -eux; \
    mkdir /dummy-certs && cd /dummy-certs/; \
    apt update && apt -y install curl openssl; \
    curl -fL https://larsenv.github.io/NintendoCerts/WII_NWC_1_CERT.p12 -O; \
    openssl pkcs12 -in WII_NWC_1_CERT.p12 -passin pass:alpine -passout pass:alpine -out keys.txt; \
    sed -n '7,29p' keys.txt > nwc.crt; \
    sed -n '33,50p' keys.txt > nwc.key; \
    openssl genrsa -out server.key 1024; \
    printf "US\nWashington\nRedmond\nNintendo of America Inc.\nNintendo Wifi Network\n*.*.*\nca@noa.nintendo.com\n\n\n" \
      | openssl req -new -key server.key -out server.csr; \
    openssl x509 -req -in server.csr -CA nwc.crt -CAkey nwc.key -CAcreateserial -out server.crt -days 3650 -sha1 -passin pass:alpine; \
    rm -f WII_NWC_1_CERT.p12 keys.txt nwc.key nwc.srl server.csr

# ==========================================================
# Build pkmn-classic-framework and dump the gts database
# （仅用 Mono 运行导入器，不依赖 Wine/X；并将 System.Data.SQLite 切换为 Mono.Data.Sqlite）
# ==========================================================
FROM debian:11 AS builder_pkmn-classic-framework
ENV DEBIAN_FRONTEND=noninteractive

# 仅需这个文件在构建上下文
COPY veekun-pokedex.sqlite.gz /tmp/veekun-pokedex.sqlite.gz

RUN set -eux; cd /; \
  # 稳定 APT
  printf 'Acquire::Retries "5";\nAcquire::http::Timeout "30";\nAcquire::https::Timeout "30";\n' > /etc/apt/apt.conf.d/80retries; \
  apt-get update; \
  # 依赖（无 Wine）
  apt-get install -y --no-install-recommends \
    ca-certificates curl git gnupg apt-transport-https \
    mariadb-server mono-complete nuget sqlite3 xz-utils \
    libmono-sqlite4.0-cil libsqlite3-0; \
  \
  # 拉代码 + 子模块
  git clone --depth 1 https://github.com/mm201/pkmn-classic-framework.git; \
  cd /pkmn-classic-framework; \
  sed -i -e 's/git@github\.com:/https:\/\/github\.com\//g' .gitmodules; \
  git submodule update --init; \
  \
  # 强制 MySQL 连接到 127.0.0.1
  find . -type f -name "*.config" -print0 \
   | xargs -0 -n1 sed -i -e 's/connectionString="Server=gts;/connectionString="Server=127.0.0.1;/g'; \
  \
  # 修引用 & 依赖（GTS）
  cd /pkmn-classic-framework/gts; \
  sed -i -e 's/<Reference Include=\"System.Web.Entity\" \/>//g' gts.csproj; \
  nuget install System.Web.Http.Common; \
  \
  # Patch VeekunImport：去掉 ReadKey + 将 System.Data.SQLite 切为 Mono.Data.Sqlite
  cd /pkmn-classic-framework/VeekunImport; \
  sed -i -e 's/Console.ReadKey();//g' Program.cs; \
  # 命名空间替换
  sed -i -e 's/^using[[:space:]]\+System\.Data\.SQLite;/using Mono.Data.Sqlite;/' Program.cs; \
  # 类型替换
  sed -i \
    -e 's/\bSQLiteConnection\b/SqliteConnection/g' \
    -e 's/\bSQLiteCommand\b/SqliteCommand/g' \
    -e 's/\bSQLiteTransaction\b/SqliteTransaction/g' \
    -e 's/\bSQLiteDataReader\b/SqliteDataReader/g' \
    -e 's/\bSQLiteParameter\b/SqliteParameter/g' \
    Program.cs; \
  # csproj 添加对 Mono.Data.Sqlite 的引用（GAC 内置）
  grep -q 'Mono.Data.Sqlite' VeekunImport.csproj || \
    awk '1; /<ItemGroup>/ && ++n==2 {print "    <Reference Include=\"Mono.Data.Sqlite\" />"}' VeekunImport.csproj > VeekunImport.csproj.new && mv VeekunImport.csproj.new VeekunImport.csproj; \
  \
  # 还原 NuGet 包并编译（xbuild 的 deprecation 提示可忽略）
  cd /pkmn-classic-framework; \
  nuget restore || true; \
  cd VeekunImport; xbuild /p:Configuration=Debug; cd -; \
  cd gts; xbuild /p:Configuration=Release /p:OutDir=publish/; \
  \
  # 准备 pokedex.sqlite
  gzip -d /tmp/veekun-pokedex.sqlite.gz; \
  install -d -m 0755 /pkmn-classic-framework/VeekunImport/bin/Debug; \
  cp -f /tmp/veekun-pokedex.sqlite /pkmn-classic-framework/VeekunImport/bin/Debug/pokedex.sqlite; \
  \
  # 初始化 MariaDB + 导入 schema
  printf "[mysqld]\nlower_case_table_names=1\n" >> /etc/mysql/my.cnf; \
  service mariadb start; \
  for i in $(seq 1 60); do mysqladmin ping --silent && break; sleep 1; done; \
  echo "CREATE DATABASE IF NOT EXISTS gts; \
        CREATE USER IF NOT EXISTS 'gts'@'localhost' IDENTIFIED BY 'gts'; \
        GRANT ALL ON *.* TO 'gts'@'localhost'; \
        FLUSH PRIVILEGES;" | mysql --user=root; \
  test -f /pkmn-classic-framework/library/database.sql || { ls -la /pkmn-classic-framework/library; exit 1; }; \
  mysql --user=root --database=gts < /pkmn-classic-framework/library/database.sql; \
  \
  # 用 Mono 跑导入器（无 GUI、无 Wine、无 SQLite.Interop.dll）
  cd /pkmn-classic-framework/VeekunImport/bin/Debug; \
  mono VeekunImport.exe; \
  cd /; mysqldump --user=root gts > /gts_dump.sql

# =========================
# Isolate pip from runner
# =========================
FROM python:2.7.18 AS builder_twisted
RUN pip install twisted

# =========================
# pkmn-server runtime
# =========================
FROM debian:11
ARG ADMIN_USERNAME
ARG ADMIN_PASSWORD
ARG VERSION_OPENSSL
ARG VERSION_HTTPD
ENV DEBIAN_FRONTEND=noninteractive
COPY --from=builder_openssl /$VERSION_OPENSSL /$VERSION_OPENSSL
COPY --from=builder_openssl /$VERSION_HTTPD /$VERSION_HTTPD
COPY --from=builder_dummy-certs /dummy-certs /dummy-certs
COPY --from=builder_pkmn-classic-framework /pkmn-classic-framework /pkmn-classic-framework
COPY --from=builder_pkmn-classic-framework /gts_dump.sql /gts_dump.sql
COPY --from=builder_twisted /usr/local/lib/python2.7/site-packages /usr/local/lib/python2.7/dist-packages

RUN set -eux; cd /; \
    apt update && apt -y --no-install-recommends install \
        apache2 apt-transport-https ca-certificates curl git \
        libapache2-mod-mono lsb-release mono-xsp net-tools \
        python2.7 python3-software-properties software-properties-common vim; \
    ln -sf /usr/bin/python2.7 /usr/bin/python; \
    curl -fsSL https://packages.sury.org/php/apt.gpg -o /etc/apt/trusted.gpg.d/php.gpg; \
    echo "deb https://packages.sury.org/php/ $(lsb_release -sc) main" | tee /etc/apt/sources.list.d/php.list; \
    apt update && apt -y --no-install-recommends install \
        mariadb-server php7.4 php7.4-mysql php7.4-sqlite3 sqlite3; \
    \
    apt -y --no-install-recommends install build-essential; \
    cd /$VERSION_OPENSSL && make install_sw && make install_ssldirs; \
    cd /$VERSION_HTTPD && cp modules/ssl/.libs/mod_ssl.so /usr/lib/apache2/modules/; \
    rm -rf /$VERSION_OPENSSL /$VERSION_HTTPD; \
    echo "/usr/local/openssl/lib" > /etc/ld.so.conf.d/usr.local.openssl.lib.conf && ldconfig; \
    apt purge -y build-essential; \
    \
    mkdir -p /etc/apache2/certs; \
    cp /dummy-certs/server.crt /etc/apache2/certs/; \
    cp /dummy-certs/server.key /etc/apache2/certs/; \
    cp /dummy-certs/nwc.crt /etc/apache2/certs/; \
    rm -rf /dummy-certs; \
    \
    cd /var/www/; \
    git clone --depth 1 https://github.com/EnergyCube/CoWFC.git; \
    git clone --depth 1 https://github.com/EnergyCube/dwc_network_server_emulator.git; \
    sed -i -e "s/db_user = root/db_user = cowfc/g" CoWFC/Web/config.ini; \
    sed -i -e "s/db_pass = passwordhere/db_pass = cowfc/g" CoWFC/Web/config.ini; \
    sed -i -e "s/recaptcha_enabled = 1/recaptcha_enabled = 0/g" CoWFC/Web/config.ini; \
    chmod 777 CoWFC/Web/bans.log; \
    touch dwc_network_server_emulator/gpcm.db; \
    chmod -R 777 dwc_network_server_emulator/; \
    printf "\npokemondpds\t2\tRwBpAHIAYQBmAGYAZQA_\n" >> dwc_network_server_emulator/gamestats.cfg; \
    service mariadb start; \
    for i in $(seq 1 30); do mysqladmin ping --silent && break; sleep 1; done; \
    echo "CREATE DATABASE IF NOT EXISTS cowfc; CREATE USER IF NOT EXISTS 'cowfc'@'localhost' IDENTIFIED BY 'cowfc'; GRANT ALL PRIVILEGES ON *.* TO 'cowfc'@'localhost'; FLUSH PRIVILEGES;" | mysql --user=root; \
    mysql --user=root --database=cowfc < CoWFC/SQL/cowfc.sql; \
    echo "INSERT INTO users (Username, Password, Rank) VALUES ('$ADMIN_USERNAME','`/var/www/CoWFC/SQL/bcrypt-hash "$ADMIN_PASSWORD"`','1') ON DUPLICATE KEY UPDATE Password=VALUES(Password), Rank=VALUES(Rank);" | mysql --user=root --database=cowfc; \
    rm -rf html/*; \
    mv CoWFC/Web/* html/; \
    mv html/config.ini ./; \
    touch /etc/.dwc_installed; \
    \
    mkdir -p /var/www/gamestats2.gs.nintendowifi.net; \
    mv /pkmn-classic-framework/gts/publish/_PublishedWebsites/gts/* /var/www/gamestats2.gs.nintendowifi.net/; \
    echo "[mysqld]\nlower_case_table_names=1" >> /etc/mysql/my.cnf; \
    service mariadb restart; \
    for i in $(seq 1 30); do mysqladmin ping --silent && break; sleep 1; done; \
    echo "CREATE DATABASE IF NOT EXISTS gts; CREATE USER IF NOT EXISTS 'gts'@'localhost' IDENTIFIED BY 'gts'; GRANT ALL ON *.* TO 'gts'@'localhost';" | mysql --user=root; \
    mysql --user=root --database=gts < /gts_dump.sql; \
    \
    cd /etc/apache2/; \
    printf "ServerName localhost\nHttpProtocolOptions Unsafe LenientMethods Allow0.9\n" >> apache2.conf; \
    sed -i -e "s/SSLCipherSuite HIGH:!aNULL/SSLCipherSuite @SECLEVEL=0:RC4-SHA:RC4-MD5/g" mods-available/ssl.conf; \
    sed -i -e "s/SSLProtocol all -SSLv3/SSLProtocol SSLv3/g" mods-available/ssl.conf; \
    mv /var/www/dwc_network_server_emulator/tools/apache-hosts/* sites-available/; \
    sed -i -e 's/ServerAlias "nas.nintendowifi.net"//g' sites-available/nas-naswii-dls1-conntest.nintendowifi.net.conf; \
    sed -i -e 's/ServerAlias "nas.nintendowifi.net, nas.nintendowifi.net"//g' sites-available/nas-naswii-dls1-conntest.nintendowifi.net.conf; \
    printf "<VirtualHost *:80>\n        ServerAdmin webmaster@localhost\n        ServerName gamestats2.gs.nintendowifi.net\n        ServerAlias \"gamestats2.gs.nintendowifi.net, gamestats2.gs.nintendowifi.net\"\n        DocumentRoot /var/www/gamestats2.gs.nintendowifi.net\n        MonoAutoApplication disabled\n        MonoServerPath \"/usr/bin/mod-mono-server4\"\n        MonoApplications default \"/:/var/www/gamestats2.gs.nintendowifi.net\"\n        <Location />\n                SetHandler mono\n                MonoSetServerAlias default\n        </Location>\n</VirtualHost>\n" > sites-available/gamestats2.gs.nintendowifi.net.conf; \
    printf "<VirtualHost *:443>\n        ServerAdmin webmaster@localhost\n        ServerName nas.nintendowifi.net\n        ServerAlias \"nas.nintendowifi.net\"\n        ServerAlias \"nas.nintendowifi.net, nas.nintendowifi.net\"\n        ProxyPreserveHost On\n        ProxyPass / http://127.0.0.1:9000/\n        ProxyPassReverse / http://127.0.0.1:9000/\n        SSLEngine on\n        SSLCertificateFile /etc/apache2/certs/server.crt\n        SSLCertificateKeyFile /etc/apache2/certs/server.key\n        SSLCertificateChainFile /etc/apache2/certs/nwc.crt\n</VirtualHost>\n" > sites-available/nas.nintendowifi.net.conf; \
    printf "<VirtualHost *:443>\n        ServerAdmin webmaster@localhost\n        ServerName pkvldtprod.nintendo.co.jp\n        DocumentRoot /var/www/gamestats2.gs.nintendowifi.net\n        SSLEngine on\n        SSLCertificateFile /etc/apache2/certs/server.crt\n        SSLCertificateKeyFile /etc/apache2/certs/server.key\n        SSLCertificateChainFile /etc/apache2/certs/nwc.crt\n        MonoAutoApplication disabled\n        MonoServerPath \"/usr/bin/mod-mono-server4\"\n        MonoApplications default \"/:/var/www/gamestats2.gs.nintendowifi.net\"\n        <Location />\n                SetHandler mono\n                MonoSetServerAlias default\n        </Location>\n</VirtualHost>\n" > sites-available/pkvldtprod.nintendo.co.jp.conf; \
    printf "<VirtualHost _default_:443>\n\
        ServerAdmin webmaster@localhost\n\
        DocumentRoot /var/www/gamestats2.gs.nintendowifi.net\n\
        SSLEngine on\n\
        SSLCertificateFile /etc/apache2/certs/server.crt\n\
        SSLCertificateKeyFile /etc/apache2/certs/server.key\n\
        SSLCertificateChainFile /etc/apache2/certs/nwc.crt\n\
        MonoAutoApplication disabled\n\
        MonoServerPath \"/usr/bin/mod-mono-server4\"\n\
        MonoApplications default \"/:/var/www/gamestats2.gs.nintendowifi.net\"\n\
        <Location />\n\
                SetHandler mono\n\
                MonoSetServerAlias default\n\
        </Location>\n\
</VirtualHost>\n" > sites-available/000-default-ssl.conf; \
# a2ensite 000-default-ssl.conf; \
#     a2ensite pkvldtprod.nintendo.co.jp.conf; \
#     a2dismod mpm_event mod_mono_auto; \
#     a2enmod proxy proxy_http php7.4 ssl; \
#     a2ensite *.nintendowifi.net.conf; \
a2dismod mpm_event mod_mono_auto; \
a2enmod ssl proxy proxy_http php7.4; \
a2ensite *.nintendowifi.net.conf pkvldtprod.nintendo.co.jp.conf 000-default-ssl.conf; \
    printf "#!/bin/sh -eu\n\nservice mariadb start\napachectl start\ncd /var/www/dwc_network_server_emulator && python master_server.py\n" > /entrypoint.sh; \
    chmod +x /entrypoint.sh; \
    apt purge -y apt-transport-https git lsb-release net-tools python3-software-properties software-properties-common vim; \
    apt autoremove -y

RUN set -eux; cd /var/www/dwc_network_server_emulator/; \
    rm -rf dlc/*; \
    sed -i -e 's/gamecodes_return_random_file = \[.*\]/gamecodes_return_random_file = \[\]/g' nas_server.py

CMD ["/bin/sh", "/entrypoint.sh"]
