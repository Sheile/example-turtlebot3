﻿# Turtlebot3 試験環境 インストールマニュアル #2


## 構築環境(2019年3月6日現在)


# minikubeのfiwareにデバイスを登録


## 環境変数の設定


1. 環境変数の設定

   ```
   $ export CORE_ROOT=$HOME/core
   $ cd $CORE_ROOT;pwd
   ```

    - 実行結果（例）

        ```
        /home/fiware/core
        ```

   ```
   $ export PJ_ROOT=$HOME/example-turtlebot3
   $ cd $PJ_ROOT;pwd
   ```

    - 実行結果（例）

        ```
        /home/fiware/example-turtlebot3
        ```

1. 環境ファイルの実行

    ```
    $ source $CORE_ROOT/docs/minikube/env
    $ source $PJ_ROOT/docs/minikube/env
    ```


## RabbitMQのユーザ設定

1. RabbitMQのユーザ登録

    ```
    for e in $(env); do
    if [[ "${e}" =~ ^MQTT__([[:alnum:]_-]+)=([[:alnum:]_-]+)$ ]]; then
        username=${BASH_REMATCH[1]}
        password=${BASH_REMATCH[2]}
        
        kubectl exec rabbitmq-0 -- rabbitmqctl add_user ${username} ${password}
        kubectl exec rabbitmq-0 -- rabbitmqctl set_permissions -p / ${username} ".*" ".*" ".*"
    fi
    done
    ```

    - 実行結果（例）

        ```
        Adding user "iotagent" ...
        Setting permissions for user "iotagent" in vhost "/" ...
        ```

1. RabbitMQのユーザ確認

    ```
    $ kubectl exec rabbitmq-0 -- rabbitmqctl list_users
    ```

    - 実行結果（例）

        ```
        Listing users ...
        user    tags
        ros     []
        raspberrypi     []
        guest   [administrator]
        iotagent        []
        ```


## gamepad serviceの設定


1. gamepad serviceの登録

    ```
    $ TOKEN=$(cat ${CORE_ROOT}/secrets/auth-tokens.json | jq '.[0].settings.bearer_tokens[0].token' -r)
    curl -H "Authorization: bearer ${TOKEN}" -H "Fiware-Service: ${FIWARE_SERVICE}" -H "Fiware-ServicePath: ${GAMEPAD_SERVICEPATH}" -H "Content-Type: application/json" http://${HOST_IPADDR}:8080/idas/ul20/manage/iot/services/ -X POST -d @- <<__EOS__
    {
    "services": [
        {
        "apikey": "${GAMEPAD_TYPE}",
        "cbroker": "http://orion:1026",
        "resource": "/iot/d",
        "entity_type": "${GAMEPAD_TYPE}"
        }
    ]
    }
    __EOS__
    ```

    - 実行結果（例）

        ```
        {}
        ```

1. gamepad serviceの登録確認

    ```
    $ TOKEN=$(cat ${CORE_ROOT}/secrets/auth-tokens.json | jq '.[0].settings.bearer_tokens[0].token' -r)
    $ curl -sS -H "Authorization: bearer ${TOKEN}" -H "Fiware-Service: ${FIWARE_SERVICE}" -H "Fiware-Servicepath: ${GAMEPAD_SERVICEPATH}" http://${HOST_IPADDR}:8080/idas/ul20/manage/iot/services/ | jq .
    ```

    - 実行結果（例）

        ```
        {
        "count": 1,
        "services": [
            {
            "commands": [],
            "lazy": [],
            "attributes": [],
            "_id": "5c7f867b41e336001076cb96",
            "resource": "/iot/d",
            "apikey": "gamepad",
            "service": "fiwaredemo",
            "subservice": "/gamepad",
            "__v": 0,
            "static_attributes": [],
            "internal_attributes": [],
            "entity_type": "gamepad"
            }
        ]
        }
        ```


## gamepad deviceの設定

1. gamepad deviceの登録

    ```
    $ TOKEN=$(cat ${CORE_ROOT}/secrets/auth-tokens.json | jq '.[0].settings.bearer_tokens[0].token' -r)
    $ curl -H "Authorization: bearer ${TOKEN}" -H "Fiware-Service: ${FIWARE_SERVICE}" -H "Fiware-ServicePath: ${GAMEPAD_SERVICEPATH}" -H "Content-Type: application/json" http://${HOST_IPADDR}:8080/idas/ul20/manage/iot/devices/ -X POST -d @- <<__EOS__
    {
    "devices": [
        {
        "device_id": "${GAMEPAD_ID}",
        "entity_name": "${GAMEPAD_ID}",
        "entity_type": "${GAMEPAD_TYPE}",
        "timezone": "Asia/Tokyo",
        "protocol": "UL20",
        "attributes": [
            {
            "name": "button",
            "type": "string"
            }
        ],
        "transport": "AMQP"
        }
    ]
    }
    __EOS__
    ```

    - 実行結果（例）

        ```
        {}
        ```

1. idas側でgamepad deviceの登録確認

    ```
    $ TOKEN=$(cat ${CORE_ROOT}/secrets/auth-tokens.json | jq '.[0].settings.bearer_tokens[0].token' -r)
    $ curl -sS -H "Authorization: bearer ${TOKEN}" -H "Fiware-Service: ${FIWARE_SERVICE}" -H "Fiware-Servicepath: ${GAMEPAD_SERVICEPATH}" http://${HOST_IPADDR}:8080/idas/ul20/manage/iot/devices/${GAMEPAD_ID}/ | jq .
    ```

    - 実行結果（例）

        ```
        {
        "device_id": "gamepad",
        "service": "fiwaredemo",
        "service_path": "/gamepad",
        "entity_name": "gamepad",
        "entity_type": "gamepad",
        "transport": "AMQP",
        "attributes": [
            {
            "object_id": "button",
            "name": "button",
            "type": "string"
            }
        ],
        "lazy": [],
        "commands": [],
        "static_attributes": [],
        "protocol": "UL20"
        }
        ```

1. orion側でgamepad deviceの登録確認

    ```
    $ TOKEN=$(cat ${CORE_ROOT}/secrets/auth-tokens.json | jq '.[0].settings.bearer_tokens[0].token' -r)
    $ curl -sS -H "Authorization: bearer ${TOKEN}" -H "Fiware-Service: ${FIWARE_SERVICE}" -H "Fiware-Servicepath: ${GAMEPAD_SERVICEPATH}" http://${HOST_IPADDR}:8080/orion/v2/entities/${GAMEPAD_ID}/ | jq .
    ```

    - 実行結果（例）

        ```
        {
        "id": "gamepad",
        "type": "gamepad",
        "TimeInstant": {
            "type": "ISO8601",
            "value": " ",
            "metadata": {}
        },
        "button": {
            "type": "string",
            "value": " ",
            "metadata": {}
        }
        }
        ```


## cygnus-mongoをgamepad deviceの購読者として設定

1. cygnus-mongoをgamepad deviceの購読者として登録

    ```
    TOKEN=$(cat ${CORE_ROOT}/secrets/auth-tokens.json | jq '.[0].settings.bearer_tokens[0].token' -r)
    curl -i -H "Authorization: bearer ${TOKEN}" -H "Fiware-Service: ${FIWARE_SERVICE}" -H "Fiware-ServicePath: ${GAMEPAD_SERVICEPATH}" -H "Content-Type: application/json" http://${HOST_IPADDR}:8080/orion/v2/subscriptions/ -X POST -d @- <<__EOS__
    {
    "subject": {
        "entities": [{
        "idPattern": "${GAMEPAD_ID}.*",
        "type": "${GAMEPAD_TYPE}"
        }],
        "condition": {
        "attrs": ["button"]
        }
    },
    "notification": {
        "http": {
        "url": "http://cygnus-mongo:5050/notify"
        },
        "attrs": ["button"],
        "attrsFormat": "legacy"
    }
    }
    __EOS__
    ```

    - 実行結果（例）

        ```
        HTTP/1.1 201 Created
        content-length: 0
        location: /v2/subscriptions/5c7f89b8207d0e5abee03ee8
        fiware-correlator: d3a4ce4c-3fec-11e9-b8be-0242ac110013
        date: Wed, 06 Mar 2019 08:50:03 GMT
        x-envoy-upstream-service-time: 2713
        server: envoy
        ```

1. cygnus-mongoがgamepad deviceの購読者として設定されたことを確認

    ```
    $ TOKEN=$(cat ${CORE_ROOT}/secrets/auth-tokens.json | jq '.[0].settings.bearer_tokens[0].token' -r)
    $ curl -sS -H "Authorization: bearer ${TOKEN}" -H "Fiware-Service: ${FIWARE_SERVICE}" -H "Fiware-ServicePath: ${GAMEPAD_SERVICEPATH}" http://${HOST_IPADDR}:8080/orion/v2/subscriptions/ | jq .
    ```

    - 実行結果（例）

    ```
    [
    {
        "id": "5c7f89b8207d0e5abee03ee8",
        "status": "active",
        "subject": {
        "entities": [
            {
            "idPattern": "gamepad.*",
            "type": "gamepad"
            }
        ],
        "condition": {
            "attrs": [
            "button"
            ]
        }
        },
        "notification": {
        "timesSent": 1,
        "lastNotification": "2019-03-06T08:50:00.00Z",
        "attrs": [
            "button"
        ],
        "attrsFormat": "legacy",
        "http": {
            "url": "http://cygnus-mongo:5050/notify"
        }
        }
    }
    ]
    ```


## robot serviceの設定

1. robot serviceの登録

    ```
    TOKEN=$(cat ${CORE_ROOT}/secrets/auth-tokens.json | jq '.[0].settings.bearer_tokens[0].token' -r)
    curl -H "Authorization: bearer ${TOKEN}" -H "Fiware-Service: ${FIWARE_SERVICE}" -H "Fiware-ServicePath: ${ROBOT_SERVICEPATH}" -H "Content-Type: application/json" http://${HOST_IPADDR}:8080/idas/ul20/manage/iot/services/ -X POST -d @- <<__EOS__
    {
    "services": [
        {
        "apikey": "${ROBOT_TYPE}",
        "cbroker": "http://orion:1026",
        "resource": "/iot/d",
        "entity_type": "${ROBOT_TYPE}"
        }
    ]
    }
    __EOS__
    ```

    - 実行結果（例）

        ```
        {}
        ```

1. robot serviceの登録確認

    ```
    $ TOKEN=$(cat ${CORE_ROOT}/secrets/auth-tokens.json | jq '.[0].settings.bearer_tokens[0].token' -r)
    $ curl -sS -H "Authorization: bearer ${TOKEN}" -H "Fiware-Service: ${FIWARE_SERVICE}" -H "Fiware-Servicepath: ${ROBOT_SERVICEPATH}" http://${HOST_IPADDR}:8080/idas/ul20/manage/iot/services/ | jq .
    ```

    - 実行結果（例）

        ```
        {
        "count": 1,
        "services": [
            {
            "commands": [],
            "lazy": [],
            "attributes": [],
            "_id": "5c7f8a6ad20f21001193feb9",
            "resource": "/iot/d",
            "apikey": "robot",
            "service": "fiwaredemo",
            "subservice": "/robot",
            "__v": 0,
            "static_attributes": [],
            "internal_attributes": [],
            "entity_type": "robot"
            }
        ]
        }
        ```


## robot deviceの設定

1. robot deviceの登録

    ```
    $ TOKEN=$(cat ${CORE_ROOT}/secrets/auth-tokens.json | jq '.[0].settings.bearer_tokens[0].token' -r)
    $ curl -H "Authorization: bearer ${TOKEN}" -H "Fiware-Service: ${FIWARE_SERVICE}" -H "Fiware-ServicePath: ${ROBOT_SERVICEPATH}" -H "Content-Type: application/json" http://${HOST_IPADDR}:8080/idas/ul20/manage/iot/devices/ -X POST -d @- <<__EOS__
    {
    "devices": [
        {
        "device_id": "${ROBOT_ID}",
        "entity_name": "${ROBOT_ID}",
        "entity_type": "${ROBOT_TYPE}",
        "timezone": "Asia/Tokyo",
        "protocol": "UL20",
        "attributes": [
            {
            "name": "x",
            "type": "float32"
            },
            {
            "name": "y",
            "type": "float32"
            },
            {
            "name": "z",
            "type": "float32"
            },
            {
            "name": "theta",
            "type": "float32"
            },
            {
            "name": "voltage",
            "type": "float32"
            },
            {
            "name": "current",
            "type": "float32"
            },
            {
            "name": "charge",
            "type": "float32"
            },
            {
            "name": "capacity",
            "type": "float32"
            },
            {
            "name": "design_capacity",
            "type": "float32"
            },
            {
            "name": "percentage",
            "type": "float32"
            }
        ],
        "commands": [
            {
            "name": "move",
            "type": "string"
            }
        ],
        "transport": "AMQP"
        }
    ]
    }
    __EOS__
    ```

    - 実行結果（例）

        ```
        {}
        ```

1. idas側のrobot deviceの登録確認

    ```
    $ TOKEN=$(cat ${CORE_ROOT}/secrets/auth-tokens.json | jq '.[0].settings.bearer_tokens[0].token' -r)
    $ curl -sS -H "Authorization: bearer ${TOKEN}" -H "Fiware-Service: ${FIWARE_SERVICE}" -H "Fiware-Servicepath: ${ROBOT_SERVICEPATH}" http://${HOST_IPADDR}:8080/idas/ul20/manage/iot/devices/${ROBOT_ID}/ | jq .
    ```

    - 実行結果（例）

        ```
        {
        "device_id": "turtlebot3",
        "service": "fiwaredemo",
        "service_path": "/robot",
        "entity_name": "turtlebot3",
        "entity_type": "robot",
        "transport": "AMQP",
        "attributes": [
            {
            "object_id": "x",
            "name": "x",
            "type": "float32"
            },
            {
            "object_id": "y",
            "name": "y",
            "type": "float32"
            },
            {
            "object_id": "z",
            "name": "z",
            "type": "float32"
            },
            {
            "object_id": "theta",
            "name": "theta",
            "type": "float32"
            },
            {
            "object_id": "voltage",
            "name": "voltage",
            "type": "float32"
            },
            {
            "object_id": "current",
            "name": "current",
            "type": "float32"
            },
            {
            "object_id": "charge",
            "name": "charge",
            "type": "float32"
            },
            {
            "object_id": "capacity",
            "name": "capacity",
            "type": "float32"
            },
            {
            "object_id": "design_capacity",
            "name": "design_capacity",
            "type": "float32"
            },
            {
            "object_id": "percentage",
            "name": "percentage",
            "type": "float32"
            }
        ],
        "lazy": [],
        "commands": [
            {
            "object_id": "move",
            "name": "move",
            "type": "string"
            }
        ],
        "static_attributes": [],
        "protocol": "UL20"
        }
        ```

1. orion側のrobot deviceの登録確認

    ```
    $ TOKEN=$(cat ${CORE_ROOT}/secrets/auth-tokens.json | jq '.[0].settings.bearer_tokens[0].token' -r)
    $ curl -sS -H "Authorization: bearer ${TOKEN}" -H "Fiware-Service: ${FIWARE_SERVICE}" -H "Fiware-Servicepath: ${ROBOT_SERVICEPATH}" http://${HOST_IPADDR}:8080/orion/v2/entities/${ROBOT_ID}/ | jq .
    ```

    - 実行結果（例）

        ```
        {
        "id": "turtlebot3",
        "type": "robot",
        "TimeInstant": {
            "type": "ISO8601",
            "value": " ",
            "metadata": {}
        },
        "capacity": {
            "type": "float32",
            "value": " ",
            "metadata": {}
        },
        "charge": {
            "type": "float32",
            "value": " ",
            "metadata": {}
        },
        "current": {
            "type": "float32",
            "value": " ",
            "metadata": {}
        },
        "design_capacity": {
            "type": "float32",
            "value": " ",
            "metadata": {}
        },
        "move_info": {
            "type": "commandResult",
            "value": " ",
            "metadata": {}
        },
        "move_status": {
            "type": "commandStatus",
            "value": "UNKNOWN",
            "metadata": {}
        },
        "percentage": {
            "type": "float32",
            "value": " ",
            "metadata": {}
        },
        "theta": {
            "type": "float32",
            "value": " ",
            "metadata": {}
        },
        "voltage": {
            "type": "float32",
            "value": " ",
            "metadata": {}
        },
        "x": {
            "type": "float32",
            "value": " ",
            "metadata": {}
        },
        "y": {
            "type": "float32",
            "value": " ",
            "metadata": {}
        },
        "z": {
            "type": "float32",
            "value": " ",
            "metadata": {}
        },
        "move": {
            "type": "string",
            "value": "",
            "metadata": {}
        }
        }
        ```


## cygnus-mongoをrobot deviceの購読者として設定 (robot position)

1. cygnus-mongoをrobot deviceの購読者として登録 (robot position)

    ```
    $ TOKEN=$(cat ${CORE_ROOT}/secrets/auth-tokens.json | jq '.[0].settings.bearer_tokens[0].token' -r)
    $ curl -i -H "Authorization: bearer ${TOKEN}" -H "Fiware-Service: ${FIWARE_SERVICE}" -H "Fiware-ServicePath: ${ROBOT_SERVICEPATH}" -H "Content-Type: application/json" http://${HOST_IPADDR}:8080/orion/v2/subscriptions/ -X POST -d @- <<__EOS__
    {
    "subject": {
        "entities": [{
        "idPattern": "${ROBOT_ID}.*",
        "type": "${ROBOT_TYPE}"
        }],
        "condition": {
        "attrs": ["x", "y", "z", "theta", "move_status", "move_info"]
        }
    },
    "notification": {
        "http": {
        "url": "http://cygnus-mongo:5050/notify"
        },
        "attrs": ["x", "y", "z", "theta", "move_status", "move_info"],
        "attrsFormat": "legacy"
    }
    }
    __EOS__
    ```

    - 実行結果（例）

        ```
        HTTP/1.1 201 Created
        content-length: 0
        location: /v2/subscriptions/5c7f8d3976d1de49620cdc43
        fiware-correlator: ea0937f2-3fee-11e9-8bed-0242ac110011
        date: Wed, 06 Mar 2019 09:04:57 GMT
        x-envoy-upstream-service-time: 8
        server: envoy
        ```

1.  cygnus-mongoがrobot deviceの購読者として設定されたことを確認

    ```
    $ TOKEN=$(cat ${CORE_ROOT}/secrets/auth-tokens.json | jq '.[0].settings.bearer_tokens[0].token' -r)
    $ curl -sS -H "Authorization: bearer ${TOKEN}" -H "Fiware-Service: ${FIWARE_SERVICE}" -H "Fiware-ServicePath: ${ROBOT_SERVICEPATH}" http://${HOST_IPADDR}:8080/orion/v2/subscriptions/ | jq .
    ```

    - 実行結果（例）

        ```
        [
        {
            "id": "5c7f8d3976d1de49620cdc43",
            "status": "active",
            "subject": {
            "entities": [
                {
                "idPattern": "turtlebot3.*",
                "type": "robot"
                }
            ],
            "condition": {
                "attrs": [
                "x",
                "y",
                "z",
                "theta",
                "move_status",
                "move_info"
                ]
            }
            },
            "notification": {
            "timesSent": 1,
            "lastNotification": "2019-03-06T09:04:57.00Z",
            "attrs": [
                "x",
                "y",
                "z",
                "theta",
                "move_status",
                "move_info"
            ],
            "attrsFormat": "legacy",
            "http": {
                "url": "http://cygnus-mongo:5050/notify"
            },
            "lastSuccess": "2019-03-06T09:04:57.00Z"
            }
        }
        ]
        ```


## gamepadのボタンテスト

1. gamepadをエミュレーションするコマンドの作成

    ```
    $ d=$(date '+%Y-%m-%dT%H:%M:%S.%s+0900')
    $ echo "mosquitto_pub -h ${HOST_IPADDR} -p 1883 -d -u iotagent -P ${MQTT__iotagent} -t /${GAMEPAD_TYPE}/${GAMEPAD_ID}/attrs -m \"${d}|button|circle\""
    ```

    - 実行結果（例）

        ```
        mosquitto_pub -h 192.168.99.1 -p 1883 -d -u iotagent -P password_of_iotagent -t /gamepad/gamepad/attrs -m "2019-03-07T13:53:29.1551934409+0900|button|circle"
        ```

1. エミュレーションコマンドの受信待機

    ```
    $ mosquitto_sub -h ${HOST_IPADDR} -p 1883 -d -u iotagent -P ${MQTT__iotagent} -t /#
    ```

    - 実行結果（例）

        ```
        Client mosqsub/31214-roboticba sending CONNECT
        Client mosqsub/31214-roboticba received CONNACK
        Client mosqsub/31214-roboticba sending SUBSCRIBE (Mid: 1, Topic: /#, QoS: 0)

        Client mosqsub/31214-roboticba received SUBACK
        Subscribed (mid: 1): 0
        ```

1. 別ターミナルで作成したエミュレーションコマンドの実行

    ```
    $ mosquitto_pub -h 192.168.99.1 -p 1883 -d -u iotagent -P password_of_iotagent -t /gamepad/gamepad/attrs -m "2019-03-07T13:53:29.1551934409+0900|button|circle"
    ```

    - 実行結果（例）

        ```
        Client mosqsub/31214-roboticba sending CONNECT
        Client mosqsub/31214-roboticba received CONNACK
        Client mosqsub/31214-roboticba sending SUBSCRIBE (Mid: 1, Topic: /#, QoS: 0)
        ```

1. 受信待機側の端末で下記が表示されていることを確認

    ```
    Client mosqpub/31264-roboticba sending PUBLISH (d0, q0, r0, m1, '/gamepad/gamepad/attrs', ... (49 bytes))
    ```

1. gamepad entityの確認

    ```
    $ TOKEN=$(cat ${CORE_ROOT}/secrets/auth-tokens.json | jq '.[0].settings.bearer_tokens[0].token' -r)
    $ curl -sS -H "Authorization: bearer ${TOKEN}" -H "Fiware-Service: ${FIWARE_SERVICE}" -H "Fiware-ServicePath: ${GAMEPAD_SERVICEPATH}" http://${HOST_IPADDR}:8080/orion/v2/entities/${GAMEPAD_ID}/ | jq .
    ```

    - 実行結果（例）

        ```
        {
		  "id": "gamepad",
		  "type": "gamepad",
		  "TimeInstant": {
		    "type": "ISO8601",
		    "value": "2019-02-26T17:22:51.1551169371+0900",
		    "metadata": {}
		  },
		  "button": {
		    "type": "string",
		    "value": "circle",
		    "metadata": {
		      "TimeInstant": {
		        "type": "ISO8601",
		        "value": "2019-02-26T17:22:51.1551169371+0900"
		      }
		    }
		  }
		}
        ```

1. cygnus-mongoの確認

    ```
    $ kubectl exec mongodb-0 -c mongodb-replicaset -- mongo sth_${FIWARE_SERVICE} --eval "db.getCollection(\"sth_${GAMEPAD_SERVICEPATH}_${GAMEPAD_ID}_${GAMEPAD_TYPE}\").find().sort({recvTime: -1})"
    ```

    - 実行結果（例）

        ```
        MongoDB shell version v4.0.6
        connecting to: mongodb://127.0.0.1:27017/sth_fiwaredemo?gssapiServiceName=mongodb
        Implicit session: session { "id" : UUID("850ebd6b-edfc-46bc-b785-07ecdf18d2f8") }
        MongoDB server version: 4.0.6
        { "_id" : ObjectId("5c946ba198f9e000117f0c67"), "recvTime" : ISODate("2019-02-26T08:22:51.155Z"), "attrName" : "button", "attrType" : "string", "attrValue" : "circle" }
        ```


## robotのx、y、z、thetaテスト

1. robotをエミュレーションするコマンドの作成

    ```
    $ d=$(date '+%Y-%m-%dT%H:%M:%S.%s+0900')
    $ echo "mosquitto_pub -h ${HOST_IPADDR} -p 1883 -d -u iotagent -P ${MQTT__iotagent} -t /${ROBOT_TYPE}/${ROBOT_ID}/attrs -m \"${d}|x|0.1|y|0.2|z|0.3|theta|0.4\""
    ```

    - 実行結果（例）

        ```
        mosquitto_pub -h 192.168.99.1 -p 1883 -d -u iotagent -P password_of_iotagent -t /robot/turtlebot3/attrs -m "2019-03-07T14:18:18.1551935898+0900|x|0.1|y|0.2|z|0.3|theta|0.4"
        ```

1. エミュレーションコマンドの受信待機

    ```
    $ mosquitto_sub -h ${HOST_IPADDR} -p 1883 -d -u iotagent -P ${MQTT__iotagent} -t /#
    ```

    - 実行結果（例）

        ```
        Client mosqsub/1414-roboticbas sending CONNECT
        Client mosqsub/1414-roboticbas received CONNACK
        Client mosqsub/1414-roboticbas sending SUBSCRIBE (Mid: 1, Topic: /#, QoS: 0)
        Client mosqsub/1414-roboticbas received SUBACK
        Subscribed (mid: 1): 0
        ```

1. 別ターミナルで作成したエミュレーションコマンドの実行

    ```
    $ mosquitto_pub -h 192.168.99.1 -p 1883 -d -u iotagent -P password_of_iotagent -t /robot/turtlebot3/attrs -m "2019-03-07T14:18:18.1551935898+0900|x|0.1|y|0.2|z|0.3|theta|0.4"
    ```

    - 実行結果（例）

        ```
        Client mosqpub/1431-roboticbas sending CONNECT
        Client mosqpub/1431-roboticbas received CONNACK
        Client mosqpub/1431-roboticbas sending PUBLISH (d0, q0, r0, m1, '/robot/turtlebot3/attrs', ... (63 bytes))
        Client mosqpub/1431-roboticbas sending DISCONNECT
        ```

1. 受信待機側の端末で下記が表示されていることを確認

    ```
    Client mosqsub/1414-roboticbas received PUBLISH (d0, q0, r0, m0, '/robot/turtlebot3/attrs', ... (63 bytes))
    2019-03-07T14:18:18.1551935898+0900|x|0.1|y|0.2|z|0.3|theta|0.4
    ```

1. robot entityの確認

    ```
    $ TOKEN=$(cat ${CORE_ROOT}/secrets/auth-tokens.json | jq '.[0].settings.bearer_tokens[0].token' -r)
    $ curl -sS -H "Authorization: bearer ${TOKEN}" -H "Fiware-Service: ${FIWARE_SERVICE}" -H "Fiware-ServicePath: ${ROBOT_SERVICEPATH}" http://${HOST_IPADDR}:8080/orion/v2/entities/${ROBOT_ID}/ | jq .
    ```

    - 実行結果（例）

        ```
        {
            "id": "turtlebot3",
            "type": "robot",
            "TimeInstant": {
            "type": "ISO8601",
            "value": "2019-03-07T14:18:18.1551935898+0900",
            "metadata": {}
            },
            "capacity": {
            "type": "float32",
            "value": " ",
            "metadata": {}
            },
            "charge": {
            "type": "float32",
            "value": " ",
            "metadata": {}
            },
            "current": {
            "type": "float32",
            "value": " ",
            "metadata": {}
            },
            "design_capacity": {
            "type": "float32",
            "value": " ",
            "metadata": {}
            },
            "move_info": {
            "type": "commandResult",
            "value": " ",
            "metadata": {}
            },
            "move_status": {
            "type": "commandStatus",
            "value": "UNKNOWN",
            "metadata": {}
            },
            "percentage": {
            "type": "float32",
            "value": " ",
            "metadata": {}
            },
            "theta": {
            "type": "float32",
            "value": "0.4",
            "metadata": {
                "TimeInstant": {
                "type": "ISO8601",
                "value": "2019-03-07T14:18:18.1551935898+0900"
                }
            }
            },
            "voltage": {
            "type": "float32",
            "value": " ",
            "metadata": {}
            },
            "x": {
            "type": "float32",
            "value": "0.1",
            "metadata": {
                "TimeInstant": {
                "type": "ISO8601",
                "value": "2019-03-07T14:18:18.1551935898+0900"
                }
            }
            },
            "y": {
            "type": "float32",
            "value": "0.2",
            "metadata": {
                "TimeInstant": {
                "type": "ISO8601",
                "value": "2019-03-07T14:18:18.1551935898+0900"
                }
            }
            },
            "z": {
            "type": "float32",
            "value": "0.3",
            "metadata": {
                "TimeInstant": {
                "type": "ISO8601",
                "value": "2019-03-07T14:18:18.1551935898+0900"
                }
            }
            },
            "move": {
            "type": "string",
            "value": "",
            "metadata": {}
            }
        }
        ```

1. cygnus-mongoの確認

    ```
    $ OP_IN='$in'
    $ kubectl exec mongodb-0 -c mongodb-replicaset -- mongo sth_${FIWARE_SERVICE} --eval "db.getCollection(\"sth_${ROBOT_SERVICEPATH}_${ROBOT_ID}_${ROBOT_TYPE}\").find().sort({recvTime: -1})"
    ```

    - 実行結果（例）

        ```
        MongoDB shell version v4.0.6
        connecting to: mongodb://127.0.0.1:27017/sth_fiwaredemo?gssapiServiceName=mongodb
        Implicit session: session { "id" : UUID("ce222934-a813-4536-b109-84467d3aa8aa") }
        MongoDB server version: 4.0.6
        { "_id" : ObjectId("5c80aa07897011001304dd1e"), "recvTime" : ISODate("2019-03-07T05:20:06.123Z"), "attrName" : "move_status", "attrType" : "commandStatus", "attrValue" : "UNKNOWN" }
        { "_id" : ObjectId("5c80aa07897011001304dd1a"), "recvTime" : ISODate("2019-03-07T05:18:18.155Z"), "attrName" : "x", "attrType" : "float32", "attrValue" : "0.1" }
        { "_id" : ObjectId("5c80aa07897011001304dd1b"), "recvTime" : ISODate("2019-03-07T05:18:18.155Z"), "attrName" : "y", "attrType" : "float32", "attrValue" : "0.2" }
        { "_id" : ObjectId("5c80aa07897011001304dd1c"), "recvTime" : ISODate("2019-03-07T05:18:18.155Z"), "attrName" : "z", "attrType" : "float32", "attrValue" : "0.3" }
        { "_id" : ObjectId("5c80aa07897011001304dd1d"), "recvTime" : ISODate("2019-03-07T05:18:18.155Z"), "attrName" : "theta", "attrType" : "float32", "attrValue" : "0.4" }
        { "_id" : ObjectId("5c7f8d3f0c76920012d54c48"), "recvTime" : ISODate("2019-03-06T09:04:57.425Z"), "attrName" : "move_status", "attrType" : "commandStatus", "attrValue" : "UNKNOWN" }
        ```


## robotのmoveテスト

1. robotをエミュレーションするコマンドの作成

    ```
    $ TOKEN=$(cat ${CORE_ROOT}/secrets/auth-tokens.json | jq '.[0].settings.bearer_tokens[0].token' -r)
    $ echo -e "curl -i -H \"Authorization: bearer ${TOKEN}\" -H \"Fiware-Service: ${FIWARE_SERVICE}\" -H \"Fiware-Servicepath: ${ROBOT_SERVICEPATH}\" -H \"Content-Type: application/json\" http://${HOST_IPADDR}:8080/orion/v2/entities/${ROBOT_ID}/attrs?type=${ROBOT_TYPE} -X PATCH -d @-<<__EOS__
    {
    \"move\": {
        \"value\": \"square\"
    }
    }
    __EOS__"
    ```

    - 実行結果（例）

        ```
        curl -i -H "Authorization: bearer 5Z9KpEAE5z3XR7ZsV5cGGeefZUOJFLv0" -H "Fiware-Service: fiwaredemo" -H "Fiware-Servicepath: /robot" -H "Content-Type: application/json" http://192.168.99.1:8080/orion/v2/entities/turtlebot3/attrs?type=robot -X PATCH -d @-<<__EOS__
        {
            "move": {
            "value": "square"
            }
        }
        __EOS__
        ```

1. エミュレーションコマンドの受信待機

    ```
    $ mosquitto_sub -h ${HOST_IPADDR} -p 1883 -d -u iotagent -P ${MQTT__iotagent} -t /#
    ```

    - 実行結果（例）

        ```
        Client mosqsub/1763-roboticbas sending CONNECT
        Client mosqsub/1763-roboticbas received CONNACK
        Client mosqsub/1763-roboticbas sending SUBSCRIBE (Mid: 1, Topic: /#, QoS: 0)
        Client mosqsub/1763-roboticbas received SUBACK
        Subscribed (mid: 1): 0
        ```

1. 別ターミナルで作成したエミュレーションコマンドの実行

    ```
    $ curl -i -H "Authorization: bearer 5Z9KpEAE5z3XR7ZsV5cGGeefZUOJFLv0" -H "Fiware-Service: fiwaredemo" -H "Fiware-Servicepath: /robot" -H "Content-Type: application/json" http://192.168.99.1:8080/orion/v2/entities/turtlebot3/attrs?type=robot -X PATCH -d @-<<__EOS__
    {
        "move": {
        "value": "square"
        }
    }
    __EOS__
    ```

    - 実行結果（例）

        ```
        HTTP/1.1 204 No Content
        content-length: 0
        fiware-correlator: 21c29ec2-4099-11e9-b8be-0242ac110013
        date: Thu, 07 Mar 2019 05:23:25 GMT
        x-envoy-upstream-service-time: 666
        server: envoy
        ```

1. 受信待機側の端末で下記が表示されていることを確認

    ```
    Client mosqsub/1763-roboticbas received PUBLISH (d0, q0, r0, m0, '/robot/turtlebot3/cmd', ... (22 bytes))
    turtlebot3@move|square
    ```

1. robot entityの確認

    ```
    $ TOKEN=$(cat ${CORE_ROOT}/secrets/auth-tokens.json | jq '.[0].settings.bearer_tokens[0].token' -r)
    $ curl -sS -H "Authorization: bearer ${TOKEN}" -H "Fiware-Service: ${FIWARE_SERVICE}" -H "Fiware-ServicePath: ${ROBOT_SERVICEPATH}" http://${HOST_IPADDR}:8080/orion/v2/entities/${ROBOT_ID}/ | jq .
    ```

    - 実行結果（例）

        ```
        {
            "id": "turtlebot3",
            "type": "robot",
            "TimeInstant": {
            "type": "ISO8601",
            "value": "2019-03-07T05:23:25.00Z",
            "metadata": {}
            },
            "capacity": {
            "type": "float32",
            "value": " ",
            "metadata": {}
            },
            "charge": {
            "type": "float32",
            "value": " ",
            "metadata": {}
            },
            "current": {
            "type": "float32",
            "value": " ",
            "metadata": {}
            },
            "design_capacity": {
            "type": "float32",
            "value": " ",
            "metadata": {}
            },
            "move_info": {
            "type": "commandResult",
            "value": " ",
            "metadata": {}
            },
            "move_status": {
            "type": "commandStatus",
            "value": "PENDING",
            "metadata": {
                "TimeInstant": {
                "type": "ISO8601",
                "value": "2019-03-07T05:23:25.767Z"
                }
            }
            },
            "percentage": {
            "type": "float32",
            "value": " ",
            "metadata": {}
            },
            "theta": {
            "type": "float32",
            "value": "0.4",
            "metadata": {
                "TimeInstant": {
                "type": "ISO8601",
                "value": "2019-03-07T14:18:18.1551935898+0900"
                }
            }
            },
            "voltage": {
            "type": "float32",
            "value": " ",
            "metadata": {}
            },
            "x": {
            "type": "float32",
            "value": "0.1",
            "metadata": {
                "TimeInstant": {
                "type": "ISO8601",
                "value": "2019-03-07T14:18:18.1551935898+0900"
                }
            }
            },
            "y": {
            "type": "float32",
            "value": "0.2",
            "metadata": {
                "TimeInstant": {
                "type": "ISO8601",
                "value": "2019-03-07T14:18:18.1551935898+0900"
                }
            }
            },
            "z": {
            "type": "float32",
            "value": "0.3",
            "metadata": {
                "TimeInstant": {
                "type": "ISO8601",
                "value": "2019-03-07T14:18:18.1551935898+0900"
                }
            }
            },
            "move": {
            "type": "string",
            "value": "",
            "metadata": {}
            }
        }
        ```

1. コマンド実行結果の送信処理をエミュレーションするコマンドの作成

    ```
    $ echo "mosquitto_pub -h ${HOST_IPADDR} -p 1883 -d -u iotagent -P ${MQTT__iotagent} -t /${ROBOT_TYPE}/${ROBOT_ID}/cmdexe -m \"${ROBOT_ID}@move|executed square command\""
    ```

    - 実行結果（例）

        ```
        mosquitto_pub -h 192.168.99.1 -p 1883 -d -u iotagent -P password_of_iotagent -t /robot/turtlebot3/cmdexe -m "turtlebot3@move|executed square command"
        ```

1. コマンド実行結果の受信待機

    ```
    $ mosquitto_sub -h ${HOST_IPADDR} -p 1883 -d -u iotagent -P ${MQTT__iotagent} -t /#
    ```

    - 実行結果（例）

        ```
        Client mosqsub/3056-roboticbas sending CONNECT
        Client mosqsub/3056-roboticbas received CONNACK
        Client mosqsub/3056-roboticbas sending SUBSCRIBE (Mid: 1, Topic: /#, QoS: 0)
        Client mosqsub/3056-roboticbas received SUBACK
        Subscribed (mid: 1): 0
        ```

1. 別ターミナルで作成したエミュレーションコマンドの実行
    ```
    $ mosquitto_pub -h 192.168.99.1 -p 1883 -d -u iotagent -P password_of_iotagent -t /robot/turtlebot3/cmdexe -m "turtlebot3@move|executed square command"
    ```

    - 実行結果（例）

        ```
        Client mosqpub/3091-roboticbas sending CONNECT
        Client mosqpub/3091-roboticbas received CONNACK
        Client mosqpub/3091-roboticbas sending PUBLISH (d0, q0, r0, m1, '/robot/turtlebot3/cmdexe', ... (39 bytes))
        Client mosqpub/3091-roboticbas sending DISCONNECT
        ```

1. 受信待機側の端末で下記が表示されていることを確認

    ```
    Client mosqsub/3056-roboticbas received PUBLISH (d0, q0, r0, m0, '/robot/turtlebot3/cmdexe', ... (39 bytes))
    ```

1. robot entityの確認

    ```
    $ TOKEN=$(cat ${CORE_ROOT}/secrets/auth-tokens.json | jq '.[0].settings.bearer_tokens[0].token' -r)
    $ curl -sS -H "Authorization: bearer ${TOKEN}" -H "Fiware-Service: ${FIWARE_SERVICE}" -H "Fiware-ServicePath: ${ROBOT_SERVICEPATH}" http://${HOST_IPADDR}:8080/orion/v2/entities/${ROBOT_ID}/ | jq .
    ```

    - 実行結果（例）

        ```
        {
            "id": "turtlebot3",
            "type": "robot",
            "TimeInstant": {
            "type": "ISO8601",
            "value": "2019-03-07T05:35:34.00Z",
            "metadata": {}
            },
            "capacity": {
            "type": "float32",
            "value": " ",
            "metadata": {}
            },
            "charge": {
            "type": "float32",
            "value": " ",
            "metadata": {}
            },
            "current": {
            "type": "float32",
            "value": " ",
            "metadata": {}
            },
            "design_capacity": {
            "type": "float32",
            "value": " ",
            "metadata": {}
            },
            "move_info": {
            "type": "commandResult",
            "value": "executed square command",
            "metadata": {
                "TimeInstant": {
                "type": "ISO8601",
                "value": "2019-03-07T05:35:34.827Z"
                }
            }
            },
            "move_status": {
            "type": "commandStatus",
            "value": "OK",
            "metadata": {
                "TimeInstant": {
                "type": "ISO8601",
                "value": "2019-03-07T05:35:34.827Z"
                }
            }
            },
            "percentage": {
            "type": "float32",
            "value": " ",
            "metadata": {}
            },
            "theta": {
            "type": "float32",
            "value": "0.4",
            "metadata": {
                "TimeInstant": {
                "type": "ISO8601",
                "value": "2019-03-07T14:18:18.1551935898+0900"
                }
            }
            },
            "voltage": {
            "type": "float32",
            "value": " ",
            "metadata": {}
            },
            "x": {
            "type": "float32",
            "value": "0.1",
            "metadata": {
                "TimeInstant": {
                "type": "ISO8601",
                "value": "2019-03-07T14:18:18.1551935898+0900"
                }
            }
            },
            "y": {
            "type": "float32",
            "value": "0.2",
            "metadata": {
                "TimeInstant": {
                "type": "ISO8601",
                "value": "2019-03-07T14:18:18.1551935898+0900"
                }
            }
            },
            "z": {
            "type": "float32",
            "value": "0.3",
            "metadata": {
                "TimeInstant": {
                "type": "ISO8601",
                "value": "2019-03-07T14:18:18.1551935898+0900"
                }
            }
            },
            "move": {
            "type": "string",
            "value": "",
            "metadata": {}
            }
        }
        ```

1. cygnus-mongoの確認

    ```
    $ OP_IN='$in'
    $ kubectl exec mongodb-0 -c mongodb-replicaset -- mongo sth_${FIWARE_SERVICE} --eval "db.getCollection(\"sth_${ROBOT_SERVICEPATH}_${ROBOT_ID}_${ROBOT_TYPE}\").find().sort({recvTime: -1})"
    ```

    - 実行結果（例）

        ```
        MongoDB shell version v4.0.6
        connecting to: mongodb://127.0.0.1:27017/sth_fiwaredemo?gssapiServiceName=mongodb
        Implicit session: session { "id" : UUID("eabba1b0-221f-4ef6-a694-a72bcc05a529") }
        MongoDB server version: 4.0.6
        { "_id" : ObjectId("5c80ada70669900016d2833b"), "recvTime" : ISODate("2019-03-07T05:35:34.827Z"), "attrName" : "move_status", "attrType" : "commandStatus", "attrValue" : "OK" }
        { "_id" : ObjectId("5c80ada70669900016d2833c"), "recvTime" : ISODate("2019-03-07T05:35:34.827Z"), "attrName" : "move_info", "attrType" : "commandResult", "attrValue" : "executed square command" }
        { "_id" : ObjectId("5c80aace0669900016d28336"), "recvTime" : ISODate("2019-03-07T05:23:25.767Z"), "attrName" : "move_status", "attrType" : "commandStatus", "attrValue" : "PENDING" }
        { "_id" : ObjectId("5c80aa07897011001304dd1e"), "recvTime" : ISODate("2019-03-07T05:20:06.123Z"), "attrName" : "move_status", "attrType" : "commandStatus", "attrValue" : "UNKNOWN" }
        { "_id" : ObjectId("5c80aa07897011001304dd1a"), "recvTime" : ISODate("2019-03-07T05:18:18.155Z"), "attrName" : "x", "attrType" : "float32", "attrValue" : "0.1" }
        { "_id" : ObjectId("5c80aa07897011001304dd1b"), "recvTime" : ISODate("2019-03-07T05:18:18.155Z"), "attrName" : "y", "attrType" : "float32", "attrValue" : "0.2" }
        { "_id" : ObjectId("5c80aa07897011001304dd1c"), "recvTime" : ISODate("2019-03-07T05:18:18.155Z"), "attrName" : "z", "attrType" : "float32", "attrValue" : "0.3" }
        { "_id" : ObjectId("5c80aa07897011001304dd1d"), "recvTime" : ISODate("2019-03-07T05:18:18.155Z"), "attrName" : "theta", "attrType" : "float32", "attrValue" : "0.4" }
        { "_id" : ObjectId("5c80aace0669900016d28332"), "recvTime" : ISODate("2019-03-07T05:18:18.155Z"), "attrName" : "x", "attrType" : "float32", "attrValue" : "0.1" }
        { "_id" : ObjectId("5c80aace0669900016d28333"), "recvTime" : ISODate("2019-03-07T05:18:18.155Z"), "attrName" : "y", "attrType" : "float32", "attrValue" : "0.2" }
        { "_id" : ObjectId("5c80aace0669900016d28334"), "recvTime" : ISODate("2019-03-07T05:18:18.155Z"), "attrName" : "z", "attrType" : "float32", "attrValue" : "0.3" }
        { "_id" : ObjectId("5c80aace0669900016d28335"), "recvTime" : ISODate("2019-03-07T05:18:18.155Z"), "attrName" : "theta", "attrType" : "float32", "attrValue" : "0.4" }
        { "_id" : ObjectId("5c80ada70669900016d28337"), "recvTime" : ISODate("2019-03-07T05:18:18.155Z"), "attrName" : "x", "attrType" : "float32", "attrValue" : "0.1" }
        { "_id" : ObjectId("5c80ada70669900016d28338"), "recvTime" : ISODate("2019-03-07T05:18:18.155Z"), "attrName" : "y", "attrType" : "float32", "attrValue" : "0.2" }
        { "_id" : ObjectId("5c80ada70669900016d28339"), "recvTime" : ISODate("2019-03-07T05:18:18.155Z"), "attrName" : "z", "attrType" : "float32", "attrValue" : "0.3" }
        { "_id" : ObjectId("5c80ada70669900016d2833a"), "recvTime" : ISODate("2019-03-07T05:18:18.155Z"), "attrName" : "theta", "attrType" : "float32", "attrValue" : "0.4" }
        { "_id" : ObjectId("5c7f8d3f0c76920012d54c48"), "recvTime" : ISODate("2019-03-06T09:04:57.425Z"), "attrName" : "move_status", "attrType" : "commandStatus", "attrValue" : "UNKNOWN" }
        ```