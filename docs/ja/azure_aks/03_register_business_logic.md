﻿# Turtlebot3 試験環境 インストールマニュアル #3


## 構築環境(2019年3月5日現在)


# Azure Kubernetes Service(AKS)のfiwareにbusiness logicを登録

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
    $ source $CORE_ROOT/docs/azure_aks/env
    $ source $PJ_ROOT/docs/azure_aks/env
    ```


## cmd-proxyの購読者登録

1. cmd-proxyの購読者登録

    ```
    $ TOKEN=$(cat ${CORE_ROOT}/secrets/auth-tokens.json | jq '.[0].settings.bearer_tokens[0].token' -r)
    $ curl -i -H "Authorization: bearer ${TOKEN}" -H "Fiware-Service: ${FIWARE_SERVICE}" -H "Fiware-ServicePath: ${GAMEPAD_SERVICEPATH}" -H "Content-Type: application/json" https://api.${DOMAIN}/orion/v2/subscriptions/ -X POST -d @- <<__EOS__
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
          "url": "http://cmd-proxy:8888/gamepad/"
        },
        "attrs": ["button"]
      }
    }
    __EOS__

    ```

    - 実行結果（例）

        ```
        HTTP/1.1 201 Created
        content-length: 0
        location: /v2/subscriptions/5c76130587d8d200d5183a54
        fiware-correlator: c8e4afbc-3a48-11e9-b967-5a27bdec5d3b
        date: Wed, 27 Feb 2019 04:33:09 GMT
        x-envoy-upstream-service-time: 3
        server: envoy
        ```

1. cmd-proxyの購読者登録確認

    ```
    $ TOKEN=$(cat ${CORE_ROOT}/secrets/auth-tokens.json | jq '.[0].settings.bearer_tokens[0].token' -r)
    $ curl -sS -H "Authorization: bearer ${TOKEN}" -H "Fiware-Service: ${FIWARE_SERVICE}" -H "Fiware-ServicePath: ${GAMEPAD_SERVICEPATH}" https://api.${DOMAIN}/orion/v2/subscriptions/ | jq .
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
            "timesSent": 4,
            "lastNotification": "2019-03-07T06:45:41.00Z",
            "attrs": [
              "button"
            ],
            "attrsFormat": "legacy",
            "http": {
              "url": "http://cygnus-mongo:5050/notify"
            },
            "lastFailure": "2019-03-07T04:55:00.00Z",
            "lastSuccess": "2019-03-07T06:45:41.00Z"
          }
        },
        {
          "id": "5c80b50b95aa76486e093459",
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
            "timesSent": 3,
            "lastNotification": "2019-03-07T06:45:41.00Z",
            "attrs": [
              "button"
            ],
            "attrsFormat": "normalized",
            "http": {
              "url": "http://cmd-proxy:8888/gamepad/"
            },
            "lastFailure": "2019-03-07T06:40:22.00Z",
            "lastSuccess": "2019-03-07T06:45:41.00Z"
          }
        }
      ]
      ```


## gamepadのボタンを押下時、robotに送信されたコマンドの確認

1. ゲームパッドをエミュレーションするコマンドの作成

    ```
    $ d=$(date '+%Y-%m-%dT%H:%M:%S.%s+0900')
    $ echo "mosquitto_pub -h mqtt.${DOMAIN} -p 8883 --cafile ${CORE_ROOT}/secrets/DST_Root_CA_X3.pem -d -u iotagent -P ${MQTT__iotagent} -t /${GAMEPAD_TYPE}/${GAMEPAD_ID}/attrs -m \"${d}|button|triangle\""
    ```

    - 実行結果（例）

      ```
      mosquitto_pub -h mqtt.example.com -p 8883 --cafile /home/fiware/core/secrets/DST_Root_CA_X3.pem -d -u iotagent -P password_of_iotagent -t /gamepad/gamepad/attrs -m "2019-02-27T13:55:23.1551243323+0900|button|triangle"
      ```

1. エミュレーションコマンドの受信待機

    ```
    $ mosquitto_sub -h mqtt.${DOMAIN} -p 8883 --cafile ${CORE_ROOT}/secrets/DST_Root_CA_X3.pem -d -u iotagent -P ${MQTT__iotagent} -t /#
    ```

    - 実行結果（例）

      ```
      Client mosqsub|27000-FIWARE-PC sending CONNECT
      Client mosqsub|27000-FIWARE-PC received CONNACK (0)
      Client mosqsub|27000-FIWARE-PC sending SUBSCRIBE (Mid: 1, Topic: /#, QoS: 0)
      Client mosqsub|27000-FIWARE-PC received SUBACK
      Subscribed (mid: 1): 0
      ```

1. 別ターミナルで作成したエミュレーションコマンドの実行

    ```
    $ mosquitto_pub -h mqtt.example.com -p 8883 --cafile /home/fiware/core/secrets/DST_Root_CA_X3.pem -d -u iotagent -P password_of_iotagent -t /gamepad/gamepad/attrs -m "2019-02-27T13:55:23.1551243323+0900|button|triangle"
    ```

    - 実行結果（例）

      ```
      Client mosqpub|27040-FIWARE-PC sending CONNECT
      Client mosqpub|27040-FIWARE-PC received CONNACK (0)
      Client mosqpub|27040-FIWARE-PC sending PUBLISH (d0, q0, r0, m1, '/gamepad/gamepad/attrs', ... (51 bytes))
      Client mosqpub|27040-FIWARE-PC sending DISCONNECT
      ```

1. 受信待機側の端末で下記が表示されていることを確認

    - 実行結果（例）

      ```
      Client mosqsub|12971-FIWARE-PC received PUBLISH (d0, q0, r0, m0, '/gamepad/gamepad/attrs', ... (51 bytes))
      2019-02-27T13:55:23.1551243323+0900|button|triangle
      Client mosqsub|6736-FIWARE-PC received PUBLISH (d0, q0, r0, m0, '/robot/turtlebot3/cmd', ... (24 bytes))
      turtlebot3@move|triangle
      ```

1. gamepad entityの確認

    ```
    $ TOKEN=$(cat ${CORE_ROOT}/secrets/auth-tokens.json | jq '.[0].settings.bearer_tokens[0].token' -r)
    $ curl -sS -H "Authorization: bearer ${TOKEN}" -H "Fiware-Service: ${FIWARE_SERVICE}" -H "Fiware-ServicePath: ${GAMEPAD_SERVICEPATH}" https://api.${DOMAIN}/orion/v2/entities/${GAMEPAD_ID}/ | jq .
    ```

    - 実行結果（例）

      ```
      {
        "id": "gamepad",
        "type": "gamepad",
        "TimeInstant": {
          "type": "ISO8601",
          "value": "2019-02-27T13:55:23.1551243323+0900",
          "metadata": {}
        },
        "button": {
          "type": "string",
          "value": "triangle",
          "metadata": {
            "TimeInstant": {
              "type": "ISO8601",
              "value": "2019-02-27T13:55:23.1551243323+0900"
            }
          }
        }
      }
      ```

1. robot entityの確認

    ```
    $ TOKEN=$(cat ${CORE_ROOT}/secrets/auth-tokens.json | jq '.[0].settings.bearer_tokens[0].token' -r)
    $ curl -sS -H "Authorization: bearer ${TOKEN}" -H "Fiware-Service: ${FIWARE_SERVICE}" -H "Fiware-ServicePath: ${ROBOT_SERVICEPATH}" https://api.${DOMAIN}/orion/v2/entities/${ROBOT_ID}/ | jq .
    ```

    - 実行結果（例）

      ```
      {
        "id": "turtlebot3",
        "type": "robot",
        "TimeInstant": {
          "type": "ISO8601",
          "value": "2019-02-26T09:00:29.00Z",
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
              "value": "2019-02-26T09:00:29.870Z"
            }
          }
        },
        "move_status": {
          "type": "commandStatus",
          "value": "PENDING",
          "metadata": {
            "TimeInstant": {
              "type": "ISO8601",
              "value": "2019-02-27T13:55:23.270Z"
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
              "value": "2019-02-26T17:41:17.1551170477+0900"
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
              "value": "2019-02-26T17:41:17.1551170477+0900"
            }
          }
        },
        "y": {
          "type": "float32",
          "value": "0.2",
          "metadata": {
            "TimeInstant": {
              "type": "ISO8601",
              "value": "2019-02-26T17:41:17.1551170477+0900"
            }
          }
        },
        "z": {
          "type": "float32",
          "value": "0.3",
          "metadata": {
            "TimeInstant": {
              "type": "ISO8601",
              "value": "2019-02-26T17:41:17.1551170477+0900"
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

1. コマンド受信結果をエミュレーションするコマンドの作成

    ```
    $ echo "mosquitto_pub -h mqtt.${DOMAIN} -p 8883 --cafile ${CORE_ROOT}/secrets/DST_Root_CA_X3.pem -d -u iotagent -P ${MQTT__iotagent} -t /${ROBOT_TYPE}/${ROBOT_ID}/cmdexe -m \"${ROBOT_ID}@move|executed triangle command\""
    ```

    - 実行結果（例）

      ```
      mosquitto_pub -h mqtt.example.com -p 8883 --cafile /home/fiware/core/secrets/DST_Root_CA_X3.pem -d -u iotagent -P password_of_iotagent -t /robot/turtlebot3/cmdexe -m "turtlebot3@move|executed triangle command"
      ```

1. エミュレーションコマンドの受信待機

    ```
    $ mosquitto_sub -h mqtt.${DOMAIN} -p 8883 --cafile ${CORE_ROOT}/secrets/DST_Root_CA_X3.pem -d -u iotagent -P ${MQTT__iotagent} -t /#
    ```

    - 実行結果（例）

      ```
      Client mosqsub|27388-FIWARE-PC sending CONNECT
      Client mosqsub|27388-FIWARE-PC received CONNACK (0)
      Client mosqsub|27388-FIWARE-PC sending SUBSCRIBE (Mid: 1, Topic: /#, QoS: 0)
      Client mosqsub|27388-FIWARE-PC received SUBACK
      Subscribed (mid: 1): 0
      ```

1. 別ターミナルで作成したエミュレーションコマンドの実行

    ```
    $ mosquitto_pub -h mqtt.example.com -p 8883 --cafile /home/fiware/core/secrets/DST_Root_CA_X3.pem -d -u iotagent -P password_of_iotagent -t /robot/turtlebot3/cmdexe -m "turtlebot3@move|executed triangle command"
    ```

    - 実行結果（例）

      ```
      Client mosqpub|27400-FIWARE-PC sending CONNECT
      Client mosqpub|27400-FIWARE-PC received CONNACK (0)
      Client mosqpub|27400-FIWARE-PC sending PUBLISH (d0, q0, r0, m1, '/robot/turtlebot3/cmdexe', ... (41 bytes))
      Client mosqpub|27400-FIWARE-PC sending DISCONNECT
      ```

1. 受信待機側の端末で下記が表示されていることを確認

      ```
      Client mosqsub|27388-FIWARE-PC received PUBLISH (d0, q0, r0, m0, '/robot/turtlebot3/cmdexe', ... (41 bytes))
      turtlebot3@move|executed triangle command
      ```

1. robot entityの確認

    ```
    $ TOKEN=$(cat ${CORE_ROOT}/secrets/auth-tokens.json | jq '.[0].settings.bearer_tokens[0].token' -r)
    $ curl -sS -H "Authorization: bearer ${TOKEN}" -H "Fiware-Service: ${FIWARE_SERVICE}" -H "Fiware-ServicePath: ${ROBOT_SERVICEPATH}" https://api.${DOMAIN}/orion/v2/entities/${ROBOT_ID}/ | jq .
    ```

    - 実行結果（例）

      ```
      {
        "id": "turtlebot3",
        "type": "robot",
        "TimeInstant": {
          "type": "ISO8601",
          "value": "2019-02-27T05:06:17.00Z",
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
          "value": "executed triangle command",
          "metadata": {
            "TimeInstant": {
              "type": "ISO8601",
              "value": "2019-02-27T05:06:17.591Z"
            }
          }
        },
        "move_status": {
          "type": "commandStatus",
          "value": "OK",
          "metadata": {
            "TimeInstant": {
              "type": "ISO8601",
              "value": "2019-02-27T05:06:17.591Z"
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
              "value": "2019-02-26T17:41:17.1551170477+0900"
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
              "value": "2019-02-26T17:41:17.1551170477+0900"
            }
          }
        },
        "y": {
          "type": "float32",
          "value": "0.2",
          "metadata": {
            "TimeInstant": {
              "type": "ISO8601",
              "value": "2019-02-26T17:41:17.1551170477+0900"
            }
          }
        },
        "z": {
          "type": "float32",
          "value": "0.3",
          "metadata": {
            "TimeInstant": {
              "type": "ISO8601",
              "value": "2019-02-26T17:41:17.1551170477+0900"
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