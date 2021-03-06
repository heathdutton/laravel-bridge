Bridge to use Laravel on AWS Lambda with Bref.

**This package is currently experimental.**

**These instructions apply to Laravel 7.**

## Installation

```bash
composer require bref/laravel-bridge
```

**Warning:** The package is not published yet, you may have to add the repository manually:

```json
    "repositories": [
        {
            "type": "vcs",
          "url": "git@github.com:brefphp/laravel-bridge.git"
        }
    ],
```

## Laravel Queues with SQS

This package lets you process jobs from SQS queues by integrating with Laravel Queues and its job system.

For example, given [a `ProcessPodcast` job](https://laravel.com/docs/7.x/queues#class-structure):

```php
<?php declare(strict_types=1);

namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class ProcessPodcast implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    /** @var int */
    private $podcastId;

    public function __construct(int $podcastId)
    {
        $this->podcastId = $podcastId;
    }

    public function handle(): void
    {
        // process the job
    }
}
```

We can dispatch this job to SQS [just like any Laravel job](https://laravel.com/docs/7.x/queues#dispatching-jobs):

```php
ProcessPodcast::dispatch($podcastId);
```

The job will be pushed to SQS. Now, instead of running the `php artisan queue:work` command, SQS will directly trigger our **handler** on AWS Lambda to process our job immediately.

### Setup

First, you need to configure [Laravel Queues](https://laravel.com/docs/7.x/queues) to use the SQS queue.

You can achieve this by setting the `QUEUE_CONNECTION` environment variable to `sqs` and configuring the rest:

```dotenv
# .env
QUEUE_CONNECTION=sqs
SQS_PREFIX=https://sqs.us-east-1.amazonaws.com/your-account-id
SQS_QUEUE=my_sqs_queue
AWS_DEFAULT_REGION=us-east-1
```

Note that on AWS Lambda, you do not need to create `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` variables: these access keys are created automatically by Lambda and available through those variables.

Create a `handler.php` file. This is the file that will handle SQS events in AWS Lambda:

```php
<?php declare(strict_types=1);

use Bref\LaravelBridge\Queue\LaravelSqsHandler;
use Illuminate\Foundation\Application;

require __DIR__ . '/vendor/autoload.php';
/** @var Application $app */
$app = require __DIR__ . '/bootstrap/app.php';

$kernel = $app->make(Illuminate\Contracts\Console\Kernel::class);
$kernel->bootstrap();

return $app->makeWith(LaravelSqsHandler::class, [
    'connection' => 'sqs',
    'queue' => getenv('SQS_QUEUE'),
]);
```

You may need to adjust the `connection` and `queue` options above if you customized the configuration in `config/queue.php`. If you are unsure, have a look [at the official Laravel documentation about connections and queues](https://laravel.com/docs/7.x/queues#connections-vs-queues).

We can now configure our handler in `serverless.yml`:

```yaml
functions:
    worker:
        handler: handler.php
        timeout: 20 # in seconds
        reservedConcurrency: 5 # max. 5 messages processed in parallel
        layers:
            - ${bref:layer.php-74}
        events:
            - sqs:
                arn: arn:aws:sqs:us-east-1:1234567890:my_sqs_queue
                # Only 1 item at a time to simplify error handling
                batchSize: 1
```

That's it! Anytime a job is pushed to the `my_sqs_queue`, SQS will invoke `handler.php` and our job will be executed.

### Differences and limitations

The SQS + Lambda integration already has a retry mechanism (with a dead letter queue). This is why those mechanisms from Laravel are not used at all. These should instead be configured on SQS (by default, jobs are retried in a loop for several days).

For those familiar with Lambda, you may know that batch processing implies that any failed job will mark all the other jobs of the batch as "failed". However, Laravel manually marks successful jobs as "completed" (i.e. those are properly deleted from SQS).
