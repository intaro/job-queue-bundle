# IntaroJobQueueBundle #

## About ##
The Job Queue Bundle allows to manage recurring, and non recurring jobs. This bundle is based on **oldsound/rabbitmq-bundle** and uses rabbitMQ server to create job queues.

## Installation ##

Require the bundle in your composer.json file:

````
{
    "require": {
        "intaro/job-queue-bundle": "dev-master",
    }
}
```

Register the bundle:

```php
// app/AppKernel.php

public function registerBundles()
{
    $bundles = array(
        new Intaro\JobQueueBundle\IntaroJobQueueBundle(),
    );
}
```

Install the bundle:

```
$ composer update intaro/job-queue-bundle
```

## Configuration ##

```yaml
intaro_job_queue:
    class:  %intaro_job_queue.job_manager.class%
    intervals:
        integration_service: { value: 3600 }
    job_timeout: 60     # timeout for executing job command
    environment: prod   # environment for executing commands
    durable: false
```

## Usage ##

Add producers and consumers to the `old_sound_rabbit_mq` section in your configuration file. You can use job_execute_service callback to execute symfony command from job`s message:

```yaml
old_sound_rabbit_mq:
    producers:
        integration:
            connection:       default
            exchange_options: {name: 'integration', type: direct}
    consumers:
        integration:
            connection:       default
            exchange_options: {name: 'integration', type: direct}
            queue_options:
                name: 'integration_main'
                routing_keys:
                    - 'integration_main'
            callback: job_execute_service
        integration_service:
            connection:       default
            exchange_options: {name: 'integration', type: direct}
            queue_options:
                name: 'integration_service'
                routing_keys:
                    - 'integration_service'
            callback: job_execute_service
```

Initiate cyclic update:

```php
    $jobManager = $container->get('job_queue_manager');
    $jobManager->addJob('acme:integration:main', 'integration_main',
        array('recurring' => true, 'interval' => 'P1D', 'startDate' => new \DateTime('00:00:00'))
        );
    $jobManager->addJob('acme:integration:service', 'integration_service',
        array('recurring' => true, 'intervalCode' => 'integration_service')
        );
```

Every day at 00:00:00 "acme:integration:main" command will be executed and "acme:integration:service" will be execudet every hour.

You can extend job_manager and override **getIntervals** method. It should return array of intervals like ("intervalCode" => seconds). It can be usefull to change intarval value in realtime.


### Available job options ###

* recurring - create recurring or non recurring job. (default - false)
* startDate - when start job executing (default - now)
* route - routing key to consumer
* intervalCode - code to find interval value for recurring job
* interval - interval for recurring job (default - 1 day)


### Commands ###

`job-queue:clear` - clears job_shedule and job_queue rabbitMQ queues.


## Consumers Script ##

Bash script to make sure that there are all required consumers are running:

```bash
#!/bin/bash

NB_TASKS=1
SYMFONY_ENV="prod"
ROOT_PATH="$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )/../"

BASE="app/console rabbitmq:consumer"
TEXT[0]="jobs"
TEXT[1]="jobs_timeout"
TEXT[2]="mobile"
TEXT[3]="integration_dictionaries"
TEXT[4]="integration_statistics"
TEXT[5]="integration_laws"
TEXT[6]="integration_service"

for text in "${TEXT[@]}"
do

    FULL_TEXT="${BASE} ${text}"
    NB_LAUNCHED=$(ps ax | grep "$FULL_TEXT" | grep -v grep | wc -l)

    TASK="php ${ROOT_PATH}${FULL_TEXT} --env=${SYMFONY_ENV}"

    for (( i=${NB_LAUNCHED}; i<${NB_TASKS}; i++ ))
    do
        echo "$(date +%c) - Launching a new consumer"
        nohup $TASK &
    done

done
```

Thanks to [Anton Babenko](http://stackoverflow.com/questions/19793839/manage-rabbitmq-consumers)