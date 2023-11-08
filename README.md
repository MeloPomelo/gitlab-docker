# Решение тестового задания 

## Развертывание GitLab

Два экземпляра GitLab очень требовательны к ресурсам, реализизовать на своей машине такое не вышло, поэтому я ипользовал один инстанс GitLab в котором создано два репозитория ``ci-test-project`` и ``sast-test-project``

![Репозитории в GitLab](/images/1.png)
_Репозитории в GitLab_

Файл _docker-compose.yml_:
```
version: '3.6'  

services:                                                    
  gitlab-dev:                                             # Создание контейнера с gitlab
    image: gitlab/gitlab-ce:16.2.8-ce.0 
    container_name: gitlab-dev
    networks:
      - dev-net
    restart: always                        
    hostname: 'localhost'         
    environment:                                         # Добавление в конфиг url инстансов giltab и container-registry
      GITLAB_OMNIBUS_CONFIG: |                           # Так как контенеры находятся в одной докер сети вместо локалхост указываем имена контенеров
        external_url 'http://gitlab-dev:8929'                   
        registry_external_url "http://gitlab-dev:5050"
    ports:                                               # Открываем порты giltab и container-registry
      - 8929:8929 
      - 5050:5050 
    volumes:                                             # Монтируем директории с данными gitlab
      - ./gitlab/config:/etc/gitlab   
      - ./gitlab/logs:/var/log/gitlab 
      - ./gitlab/data:/var/opt/gitlab   

  gitlab-runner:                                         # Создание контейнера с gitlab-runner
    image: gitlab/gitlab-runner:ubuntu-v16.4.0
    container_name: gitlab-runner    
    networks:
      - dev-net
    restart: always                            
    volumes:                                            # Монтируем директории с данными gitlab-runner
      - ./runner/config:/etc/gitlab-runner 
      - /var/run/docker.sock:/var/run/docker.sock 

  sonarqube:                                            # Создание контейнера c sonarqube
    image: sonarqube:9.9.2-community
    container_name: sonarqube
    networks:
      - dev-net
    restart: always                    
    hostname: 'localhost'    
    ports:                                             # Открываем порт для доступа к sonarqube
      - 9000:9000         
    volumes:                                           # Монтируем директории с данными sonarqube
      - './sonarqube/sonarqube_data:/opt/sonarqube/data'             
      - './sonarqube/sonarqube_extensions:/opt/sonarqube/extensions'  
      - './sonarqube/sonarqube_logs:/opt/sonarqube/logs'         

networks:                                              # Объявление докер сети с драйвером bridge 
  dev-net:
    external: true
```
# Репозиторий `ci-test-project`

В пайплайне реализовано две джобы: _build_ и _sast_sync_.

- первая собирает образ и помещает его в conatiner registry
- вторая отправляет изменения в `sast-test-project`

![Выаолнение пайплайна](/images/2.png)
_Выполнение пайплайна_

![Образ в conatiner registry](/images/3.png)
_Образ в conatiner registry_

![Синхронизация изменений](/images/4.png)
_Синхронизация изменений_

Файл _gitlab-ci.yml_:
```
stages:   
  - build  
  - test

build:        
  stage: build  
  image:       
    name: gcr.io/kaniko-project/executor:v1.14.0-debug 
    entrypoint: [""]  
  when: manual
  script:           
    - /kaniko/executor
      --context "${CI_PROJECT_DIR}"
      --dockerfile "${CI_PROJECT_DIR}/Dockerfile"
      --insecure
      --destination "${CI_REGISTRY_IMAGE}:${CI_COMMIT_TAG}"
  
sast-sync:
  stage: test
  when: manual
  before_script:
    - apk update && apk upgrade && apk add git rsync > /dev/null 
    - git config --global user.email "example@example.com"
    - git config --global user.name "example example"
  script:
    - git clone http://oauth2:${SAST_PROJECT_TOKEN}@${CI_SERVER_HOST}:8929/root/sast-test-project sast-sync
    - rsync -av ./ ./sast-sync --exclude sast-sync --exclude .git --exclude .gitlab-ci.yml > /dev/null
    - cd sast-sync
    - git add . *
    - git commit -m "sast-sync"
    - git remote set-url origin http://oauth2:${SAST_PROJECT_TOKEN}@${CI_SERVER_HOST}:8929/root/sast-test-project
    - git push -u origin HEAD 
  after_script:
    - rm -rf sast-sync
```
# Репозиторий `sast-test-project`

В пайплайне реализована джоба _sast_sync_, она сканикурет код с помощью _sonarqube_.

Для подключение к sanarqube в переменные CI/CD добавлены `SANAR_HOST` и `SONAR_TOKEN`.

![Запуск пайплайна](/images/5.png)
_Запуск пайплайна_

![Результаты сканирования](/images/6.png)
_Результаты сканирования_

Файл _gitlab-ci.yml_ в репозитории `sast-test-project`:
```
stages:   
  - sast  

variables:  
  SONAR_USER_HOME: "${CI_PROJECT_DIR}/.sonar" 
  GIT_DEPTH: "0"                                  

sonarqube-check:  
  stage: sast    
  image:          
    name: sonarsource/sonar-scanner-cli:5.0.1   
    entrypoint: [""]                      
  cache:                  
    paths:             
      - .sonar/cache  
  when: manual     
  script:                 
    - git clone ${CI_REPOSITORY_URL} sast_repo      
    - sonar-scanner -Dsonar.sources=sast_repo/app  
  after_script:
    - rm -rf sast_repo

```