# **Задача 1**

*Не ожидаем production-ready решения. Сделайте, как кажется правильным, опишите процесс поиска и принятые решения.*

Опишите решение для веб-приложения в kubernetes в виде yaml-манифеста. Оставляйте в коде комментарии по принятым решениям. Есть следующие вводные:

1. у нас мультизональный кластер (три зоны), в котором пять нод
2. приложение требует около 5-10 секунд для инициализации
3. по результатам нагрузочного теста известно, что 4 пода справляются с пиковой нагрузкой
4. на первые запросы приложению требуется значительно больше ресурсов CPU, в дальнейшем потребление ровное в районе 0.1 CPU. По памяти всегда “ровно” в районе 128M memory
5. приложение имеет дневной цикл по нагрузке – ночью запросов на порядки меньше, пик – днём
6. хотим максимально отказоустойчивый deployment
7. хотим минимального потребления ресурсов от этого deployment’а

------

## **Мое решение:**

Разберу задание по пунктам:

1. мульти зональный Кластер - 5 нод

   значит нужно задать правило, чтобы раскидывать ноды на разные зоны
   Это можно реализовать либо через Kubelet (он на узле саморегистрируется), либо в ручную добавить на API серевере. Приму 

2. Для иницализации нужно 5-10 секунд 

   это нормальный показатель, значит все что менее 10 секунд на ответ не является ошибкой, это нужно указать для мониторинга
   Перед первой пробой нужно подождать не менее 10 секунд.

3. 4 подов хватает для пиковой нагрузки

   значит максимальное общее количество подов не должно превышать это значение. В HPA нужно указать минимальное и максимальное кол-во клонов. (1 и 4) 

4. первые запросы -пик, нужно много CPU, дальше 0.1, память постоянная 128Mi

   это значит что request (memory: 128M; cpu: 0.1) limits (memory: 150М; cpu: 1) и при использования CPU на 50% буддет создаваться новый под (чтобы уменьшать нагрузку на процессор в моменты нарастания количества запросов к приложению, но втоже время давать достаточно мощности для разворачивания пода и первые запросы к приложению). 

5. 2 цикла нагрузки

   в моменты когда нагрузка будет падать балансировщик уберет лишние ресурсы

6. отказоустойчивый кластер

   - в случае отказа ноды

     настроить политику распределения подов в случае выхода из строя ноды. распределять поды и по узлам и по зонам доступности [Pod Topology Spread Constraints](https://kubernetes.io/docs/concepts/workloads/pods/pod-topology-spread-constraints/) 

   - **обновление kubernetes**ыыыыыыыыыыыыыыыыыыыыыыыыыыыыыыыыыыы

     при обновлении автоматом перезапускаются ноды, а с ними и поды. Чтобы не случилось одновременный перезапуск подов (который может привести к недоступности приложения), нужно настроить число одновременно недоступых подов и обеспечить соблюдение SLA - использовать - [Pod Disruption Budget](https://kubernetes.io/docs/tasks/run-application/configure-pdb/) 

   - Сбой DNS

     увеличим отказоустойчивость кластера путем обращения нодов к сервису кластера через локальный кэш - NodeLocal DNSCache

     ------

   Это нужно узнать как сделать:

   - отключить автообновление
   - региональный тип мастера
   - явная политика распределения трафика LoadBalancer через  параметр [externalTrafficPolicy](https://cloud.yandex.ru/docs/managed-kubernetes/operations/create-load-balancer#advanced) (тарфик попадал на прямую на нужную ноду)
   - разворачивать deployment в нескольких экземплярах
   - отслеживать через  [readiness-проб](https://kubernetes.io/ru/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes) когда приложение временно не может обслуживаться 

7. минимальное потребление ресурсов от деплоймента

   постарался рационально использовать ресурсы.

