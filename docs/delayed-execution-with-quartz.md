# Delayed execution with php-quartz
  
The tasks could be paused and reused in future. 
For example you may want to do some activitis in two days. 
It could be a check that user finished a registration, if not we should a notification email.
Here I suppose you have installed quartz as a library or as a service.
You have remote scheduler setup and working.
 

## Process engine configuration

Here's how your behavior could look like. 
At execution time it creates a trigger and sends it to quartz. Once it is done it tells the process engigne to wait.
The signal is execute when the quarzt deicideds the time has come. 
 
```php
<?php
namespace App\Pvm\Behavior;

use App\Async\Topics;
use App\JobStatus;
use App\Model\Job;
use App\Model\JobResult;
use App\Model\Process;
use Formapro\Pvm\Behavior;
use Formapro\Pvm\Exception\InterruptExecutionException;
use Formapro\Pvm\Exception\WaitExecutionException;
use Formapro\Pvm\SignalBehavior;
use Formapro\Pvm\Token;
use Quartz\App\EnqueueResponseJob;
use Quartz\App\RemoteScheduler;
use Quartz\Core\JobBuilder;
use Quartz\Core\SimpleScheduleBuilder;
use Quartz\Core\TriggerBuilder;
use Formapro\Pvm\Enqueue\HandleAsyncTransitionProcessor;

class TwoDaysDelayBehavior implements Behavior, SignalBehavior
{
    /**
     * @var RemoteScheduler
     */
    private $remoteScheduler;

    /**
     * @param RemoteScheduler $remoteScheduler
     */
    public function __construct(RemoteScheduler $remoteScheduler) 
    {
        $this->remoteScheduler = $remoteScheduler;
    }

    /**
     * {@inheritdoc}
     */
    public function execute(Token $token)
    {
        $job = JobBuilder::newJob(EnqueueResponseJob::class)->build();
        
        $trigger = TriggerBuilder::newTrigger()
            ->forJobDetail($job)
            ->withSchedule(SimpleScheduleBuilder::simpleSchedule()->repeatForever())
            ->setJobData([
                'topic' => Topics::PVM_HANDLE_ASYNC_TRANSITION,
                'process' => $token->getProcess()->getId(),
                'token' => $token->getId(),
            ])
            ->startAt(new \DateTime('now + 30 seconds'))
            ->build();

        $this->remoteScheduler->scheduleJob($trigger, $job);

        throw new WaitExecutionException;
    }

    /**
     * {@inheritdoc}
     */
    public function signal(Token $token)
    {
        // two days have passed, now we are ready to do what we are supposed to do in two days.
    }
}
```

[Back](../README.md)