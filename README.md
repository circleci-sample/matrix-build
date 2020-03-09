# matrix-build

## CircleCI マトリックスビルド

CircleCIでは、マトリックスビルドはサポートされていませんが、実現するためのいくつかの方法があります。

node v8、v10、v12、それぞれの実行環境でライブラリのインストール、テスト、ビルドを行い、すべてに成功した場合のみ、ビルドした成果物をNode v10でデプロイしたいとします。また、テストとビルドは互いに依存しないが、インストールに依存するとします。

### それぞれの実行環境ごとに、ジョブを定義する

次の図のように、実行環境(Nodeバージョン)ごとにジョブを定義することができます。

![image](https://user-images.githubusercontent.com/15242484/75761270-bfe06d80-5d73-11ea-8850-631a9a77c5b1.png)

```yaml
version: 2.1

jobs:
  install_deps:
    docker:
      - image: circleci/node:10
    steps:
      - checkout
      - restore_cache:
          keys:
            - v1-npm-{{ .Branch }}-{{ checksum "package-lock.json" }}
            - v1-npm-{{ .Branch }}-
            - v1-npm-
      - run: npm install
      - save_cache:
          paths:
            - node_modules
          key: v1-npm-{{ .Branch }}-{{ checksum "package-lock.json" }}
      - persist_to_workspace:
          root: ./
          paths:
            - node_modules

  "node-8":
    docker:
      - image: circleci/node:8
    steps:
      - checkout
      - attach_workspace:
          at: ./
      - run: npm test
      - run: npm run build

  "node-10":
    docker:
      - image: circleci/node:10
    steps:
      - checkout
      - attach_workspace:
          at: ./
      - run: npm test
      - run: npm run build
      - persist_to_workspace: # Node v10でビルドした成果物を永続化
          root: ./
          paths:
            - lib/
            - node_modules

  "node-12":
    docker:
      - image: circleci/node:12
    steps:
      - checkout
      - attach_workspace:
          at: ./
      - run: npm test
      - run: npm run build

  deploy:
    docker:
      - image: circleci/node:10
    steps:
      - checkout
      - attach_workspace:
          at: ./
      - run: npm run deploy

workflows:
  version: 2
  matrix:
    jobs:
      - install_deps
      - "node-8":
          requires:
            - install_deps
      - "node-10":
          requires:
            - install_deps
      - "node-12":
          requires:
            - install_deps
      - deploy:
          requires:
            - "node-8"
            - "node-10"
            - "node-12"
```

`test`と`build`が直列に定義されていることが気になるところですが、テストとビルドが依存していないからと言って別ジョブに分けて定義したいという欲がでてくると、`node-8-test`、`node-8-build`、`node-10-test`…をそれぞれ定義しなければならなくなるため、ちょっと厳しくなります。

![image](https://user-images.githubusercontent.com/15242484/75865074-ebc82580-5e3d-11ea-8213-49300e31a0cc.png)

```yaml
version: 2.1

jobs:
  (略)

  "node-8-test":
    docker:
      - image: circleci/node:8
    steps:
      - checkout
      - attach_workspace:
          at: ./
      - run: npm test

  "node-8-build":
    docker:
      - image: circleci/node:8
    steps:
      - checkout
      - attach_workspace:
          at: ./
      - run: npm run build

  "node-10-test":
    docker:
      - image: circleci/node:10
    steps:
      - checkout
      - attach_workspace:
          at: ./
      - run: npm test

  "node-10-build":
    docker:
      - image: circleci/node:10
    steps:
      - checkout
      - attach_workspace:
          at: ./
      - run: npm run build
      - persist_to_workspace:
          root: ./
          paths:
            - lib
            - node_modules

  "node-12-test":
    docker:
      - image: circleci/node:12
    steps:
      - install_deps
      - checkout
      - attach_workspace:
          at: ./
      - run: npm test

  "node-12-build":
    docker:
      - image: circleci/node:12
    steps:
      - checkout
      - attach_workspace:
          at: ./
      - run: npm run build

  deploy:
    docker:
      - image: circleci/node:10
    steps:
      - checkout
      - attach_workspace:
          at: ./
      - run: npm run deploy

workflows:
  version: 2
  matrix:
    jobs:
      - "node-8-test"
      - "node-8-build"
      - "node-10-test"
      - "node-10-build"
      - "node-12-test"
      - "node-12-build"
      - deploy:
          requires:
            - "node-8-test"
            - "node-8-build"
            - "node-10-test"
            - "node-10-build"
            - "node-12-test"
            - "node-12-build"
```

この場合、パラメータを使ってジョブコンフィグをまとめることができます。

### ジョブパラメータで管理する 

パラメータを使えば、テスト・ビルドのジョブは使い回すことができるため、設定ファイルがすっきりします。ただし、これでも手動で値を入れる必要があるため、2軸のマトリックス(言語バージョン×OSなど)はうまく管理できる気がしません。

```yaml
version: 2.1

executor-template: &executor-template
  parameters:
    tag:
      default: "10"
      type: string
    persist_built_files:
      type: boolean
      default: false
  docker:
    - image: circleci/node:<< parameters.tag >>

jobs:
  install_deps:
    <<: *executor-template
    steps:
      - checkout
      - restore_cache:
          keys:
            - v1-npm-{{ .Branch }}-{{ checksum "package-lock.json" }}
            - v1-npm-{{ .Branch }}-
            - v1-npm-
      - run: npm install
      - save_cache:
          paths:
            - node_modules
          key: v1-npm-{{ .Branch }}-{{ checksum "package-lock.json" }}
      - persist_to_workspace:
          root: ./
          paths:
            - node_modules

  test:
    <<: *executor-template
    steps:
      - checkout
      - attach_workspace:
          at: ./
      - run: npm run test

  build:
    <<: *executor-template
    steps:
      - checkout
      - attach_workspace:
          at: ./
      - run: npm run build
      - when:
          condition: << parameters.persist_built_files >>
          steps:
            - persist_to_workspace:
                root: ./
                paths:
                  - node_modules
                  - lib

  deploy:
    <<: *executor-template
    steps:
      - checkout
      - attach_workspace:
          at: ./
      - run: npm run deploy

workflows:
  version: 2
  matrix:
    jobs:
      - install_deps

      - test:
          name: node-8-test
          tag: "8"
          requires:
            - install_deps

      - test:
          name: node-10-test
          tag: "10"
          requires:
            - install_deps

      - test:
          name: node-12-test
          tag: "12"
          requires:
            - install_deps

      - build:
          name: node-8-build
          tag: "8"
          requires:
            - install_deps

      - build:
          name: node-10-build
          tag: "10"
          persist_built_files: true # Node v10でビルドした成果物を永続化するためのパラメータ
          requires:
            - install_deps

      - build:
          name: node-12-build
          tag: "12"
          requires:
            - install_deps

      - deploy:
          tag: "10"
          requires:
            - node-8-test
            - node-8-build
            - node-10-test
            - node-10-build
            - node-12-test
            - node-12-build
```

### パイプラインパラメータで管理する 

少しトリッキーですが、パイプラインパラメータを使えば、ワークフロー全体にパラメータを渡すことができるため、より少ない記述量で実現できます。この設定ファイルを使用するには、[APIトークン](https://circleci.com/docs/ja/2.0/managing-api-tokens/)(`CIRCLE_TOKEN`)が必要です。

(パラメータを使った動的なジョブ名は前からできたっけ？)

```yaml
parameters:
  tag:
    default: '10'
    type: string
  run-main-workflow:
    default: false
    type: boolean
  run-deploy-job:
    default: false
    type: boolean

executors:
  node:
    docker:
      - image: circleci/node:<< pipeline.parameters.tag >>

version: 2.1
jobs:
  install_deps:
    executor: node
    steps:
      - checkout
      - restore_cache:
          keys:
            - v1-npm-{{ .Branch }}-{{ checksum "package-lock.json" }}
            - v1-npm-{{ .Branch }}-
            - v1-npm-
      - run: npm install
      - save_cache:
          paths:
            - node_modules
          key: v1-npm-{{ .Branch }}-{{ checksum "package-lock.json" }}
      - persist_to_workspace:
          root: ./
          paths:
            - node_modules

  test:
    executor: node
    steps:
      - checkout
      - attach_workspace:
          at: ./
      - run: npm run test

  build:
    executor: node

    steps:
      - checkout
      - attach_workspace:
          at: ./
      - run: npm run build

      - when:
          condition: << pipeline.parameters.run-deploy-job >>
          steps:
            - persist_to_workspace:
                root: ./
                paths:
                  - lib
                  - node_modules

  deploy:
    docker:
      - image: circleci/node:<< pipeline.parameters.tag >>
    steps:
      - when:
          condition: << pipeline.parameters.run-deploy-job >>
          steps:
            - checkout
            - attach_workspace:
                at: ./
            - run: npm run deploy
      - unless:
          condition: << pipeline.parameters.run-deploy-job >>
          steps:
            - run: echo "No deployment on Node v<< pipeline.parameters.tag >>"

  trigger-main-workflows:
    machine:
      image: ubuntu-1604:201903-01
    parameters:
      deploy-node-version:
        default: '10'
        type: string
    steps:
      - run:
          name: Trigger main worflow
          command: |
            VCS_TYPE=$(echo ${CIRCLE_BUILD_URL} | cut -d '/' -f 4)

            for NODE_VERSION in 8 10 12
            do
                PIIPELINE_PARAM_MAP="{\"run-main-workflow\": true, \"tag\":\"$NODE_VERSION\"}"
                if [ "$NODE_VERSION" = << parameters.deploy-node-version >> ]
                then
                    PIIPELINE_PARAM_MAP="{\"run-main-workflow\": true, \"tag\":\"$NODE_VERSION\", \"run-deploy-job\": true}"
                fi

                curl -u ${CIRCLE_TOKEN}: -X POST --header "Content-Type: application/json" -d "{
                  \"branch\": \"${CIRCLE_BRANCH}\",
                  \"parameters\": ${PIIPELINE_PARAM_MAP}
                }" "https://circleci.com/api/v2/project/${VCS_TYPE}/${CIRCLE_PROJECT_USERNAME}/${CIRCLE_PROJECT_REPONAME}/pipeline"
            done

workflows:
  version: 2
  matrix:
    unless: << pipeline.parameters.run-main-workflow >>
    jobs:
      - trigger-main-workflows:
          deploy-node-version: "10"

  main:
    when: << pipeline.parameters.run-main-workflow >>
    jobs:
      - build:
          name: build-<< pipeline.parameters.tag >>
      - test:
          name: test-<< pipeline.parameters.tag >>
      - deploy:
          requires:
            - build-<< pipeline.parameters.tag >>
            - test-<< pipeline.parameters.tag >>
```

`trigger-jobs`ワークフローが、Node v8, v10, v12の実行環境において、それぞれ`main`ワークフローをトリガーします。

Nodeバージョン×プラットフォームといったマトリックスビルドも可能です。

[全文]()

```yaml
(略)
  trigger-main-workflows:
    machine:
      image: ubuntu-1604:201903-01
    parameters:
      deploy-node-version:
        default: '10'
        type: string
      deploy-platform:
        default: linux
        type: string
    steps:
      - run:
          name: Trigger main worflow
          command: |
            VCS_TYPE=$(echo ${CIRCLE_BUILD_URL} | cut -d '/' -f 4)

            for NODE_VERSION in 8 10 12
            do
                for PLATFORM in linux windows
                do
                    PIIPELINE_PARAM_MAP="{\"run-main-workflow\": true, \"tag\":\"$NODE_VERSION\", \"platform\":\"$PLATFORM\"}"
                    if [ "$NODE_VERSION" = "<< parameters.deploy-node-version >>" ] && [ "$PLATFORM" = "<< parameters.deploy-platform >>" ]
                    then
                        PIIPELINE_PARAM_MAP="{\"run-main-workflow\": true, \"tag\":\"$NODE_VERSION\", \"platform\":\"$PLATFORM\", \"run-deploy-job\": true}"
                    fi
                    curl -u ${CIRCLE_TOKEN}: -X POST --header "Content-Type: application/json" -d "{
                      \"branch\": \"${CIRCLE_BRANCH}\",
                      \"parameters\": ${PIIPELINE_PARAM_MAP}
                    }" "https://circleci.com/api/v2/project/${VCS_TYPE}/${CIRCLE_PROJECT_USERNAME}/${CIRCLE_PROJECT_REPONAME}/pipeline"
                done
            done
```

ただし、APIを叩く必要があるため、これでもまだ他のCIでできるようなマトリックスビルドと比べれば、やぼったさは否めません。また、デプロイジョブをすべてのワークフローが成功した場合のみ実行するしくみ(ファンイン)にすることが非常に難しく、APIを使ってビルド結果をポーリングするか、承認ジョブを使って目視で確認してから手動で起動にするしかなさそうです。

([ファンアウト・ファンインについて](https://circleci.com/docs/ja/2.0/workflows/#%E3%83%95%E3%82%A1%E3%83%B3%E3%82%A4%E3%83%B3%E3%83%95%E3%82%A1%E3%83%B3%E3%82%A2%E3%82%A6%E3%83%88%E3%81%AE-workflow-%E3%81%AE%E4%BE%8B))

[RFC: Matrix Jobs syntax](https://discuss.circleci.com/t/rfc-matrix-jobs-syntax/34468)を見ると、マトリックスビルドの開発が今まさに行われているようなので、正式サポートされるまではファンアウトだけならジョブパラメータ、ファンインが必要ならパイプラインパラメータが良さそうかな。
