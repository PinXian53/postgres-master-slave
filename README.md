## 功能
使用 docker compose 建立 postgres master slave 環境
> 內容參考：https://medium.com/@eremeykin/how-to-setup-single-primary-postgresql-replication-with-docker-compose-98c48f233bbf

## 指令
- 啟動服務
    ```shell
    docker compose -f ./docker/docker-compose.yml -p postgres-master-slave up -d
    ```
- 停止服務並刪除資料
    ```shell
    docker-compose -p postgres-master-slave down -v
    ```

## 內容說明
- 00_init.sql
    > 初始化資料庫時執行的 SQL
    ```sql
    -- 創建使用者，賦予資料複寫權限，及設定密碼
    CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'replicator_password';
    -- 創建物理複製槽
    --   用於確保 postgres 保留必要的寫前日誌（WAL）記錄，直到從節點成功處理了這些變更
    --   這樣可以防止主節點上所需的 WAL 記錄被過早刪除，確保從節點能夠順利同步。
    SELECT pg_create_physical_replication_slot('replication_slot');
    ```

- docker-compose.yml
  > docker compose 腳本 (
  > 提醒：master 使用 5432 port, slave 使用 5433 port
  ```yaml
  version: '3.8'
  services:
    postgres_master:
       image: postgres:15.8
       ports:
         - "5432:5432"
       # 指定容器內運行服務的 Linux 用戶
       user: postgres
       environment:
         POSTGRES_DB: mydatabase
         POSTGRES_USER: postgres
         POSTGRES_PASSWORD: postgres
         # 將主機身份驗證設為 scram-sha-256，並允許所有主機進行複製操作
         POSTGRES_HOST_AUTH_METHOD: "scram-sha-256\nhost replication all 0.0.0.0/0 md5"
         # 設定初始化數據庫時的認證方式
         POSTGRES_INITDB_ARGS: "--auth-host=scram-sha-256"
       # wal_level=replica：設置寫前日誌（WAL）的級別為 replica，以便支持複製
       # hot_standby=on：允許從節點在複製時處於熱備狀態，可以處理查詢
       # max_wal_senders=10 和 max_replication_slots=10：設置允許最多 10 個複製連接和 10 個複製槽
       # hot_standby_feedback=on：確保主節點不會過早地回收 WAL 記錄，避免從節點遇到複製延遲
       command: |
         postgres
         -c wal_level=replica
         -c hot_standby=on
         -c max_wal_senders=10
         -c max_replication_slots=10
         -c hot_standby_feedback=on
       volumes:
         - ./00_init.sql:/docker-entrypoint-initdb.d/00_init.sql
         - master_data:/var/lib/postgresql/data
    postgres_slave:
      image: postgres:15.8
      ports:
        - "5433:5432"
      user: postgres
      environment:
        POSTGRES_DB: mydatabase
        POSTGRES_USER: postgres
        POSTGRES_PASSWORD: postgres
        # 指定連接主節點所用的複製使用者和密碼 (00_init.sql 內建立的使用者)
        PGUSER: replicator
        PGPASSWORD: replicator_password
      # 使用 pg_basebackup 命令從主節點創建基礎備份，並設置參數：
      #   --pgdata=/var/lib/postgresql/data：備份存儲在從節點的數據目錄中。
      #   -R：自動創建 recovery.conf 文件，這樣從節點啟動時會以複製模式連接主節點。
      #   --slot=replication_slot：指定在主節點上使用的複製槽，以確保 WAL 記錄保留。
      #   --host=postgres_master 和 --port=5432：指定主節點的地址和端口。
      # 執行完成後，將數據目錄的權限設為 0700，以確保只有數據庫進程能夠讀寫這些文件
      # 最後啟動 PostgreSQL 服務
      command: |
        bash -c "
        until pg_basebackup --pgdata=/var/lib/postgresql/data -R --slot=replication_slot --host=postgres_master --port=5432
        do
        echo 'Waiting for primary to connect...'
        sleep 1s
        done
        echo 'Backup done, starting replica...'
        chmod 0700 /var/lib/postgresql/data
        postgres
        "
      volumes:
        - slave_data:/var/lib/postgresql/data
      depends_on:
        - postgres_master
  
  volumes:
    master_data:
    slave_data:
  ```
