### When things take longer than you thought: Moving workloads to background processes

Drupalcon Lille - October 2023

---

### Pablo López Escobés

Drupal developer @ Lullabot

[@plopesc](https://www.drupal.org/u/plopesc)

---

## What is this about?

Efficiency and Responsiveness

Optimizing Resource Usage

Scalability and Concurrency

Note:
* Explanation:

  * When processing a web request, some tasks may be time-consuming or resource-intensive.
  * Performing these tasks directly can slow down the response time, impacting user experience.
* Solution:

  * Delegating such tasks to background processes ensures the web server remains responsive and efficient in handling primary request processing.
  * Background processes handle secondary tasks without delaying the initial response to the user.
* Explanation:

  * Background processes can leverage additional server resources without affecting the main request handling.
  * This prevents resource exhaustion and maintains system stability.
* Solution:

  * Delegating secondary tasks to separate processes allows for efficient resource management and improved overall system performance.

* Explanation:

  * As user traffic increases, concurrency becomes crucial for effective web application performance.
  * Blocking the main thread with time-consuming tasks reduces the application's ability to handle multiple requests concurrently.
* Solution:

  * Offloading secondary tasks allows for scaling and handling more requests concurrently, providing a smoother and more responsive user experience even during peak traffic times.

---

**By delegating secondary tasks to background processes, we enhance the efficiency, responsiveness, and scalability of our web application, ensuring optimal performance and a seamless user interaction.**

---

## Criteria

Critical vs. Non-Critical

Simple vs. Complex

CPU vs. I/O-bound

User Direct Impact vs. User Indirect Impact

Urgent vs. Non-Urgent

---

## Candidate tasks

EMail/Push Notifications, File Processing

Report Generation, Data Import/Export

Image Processing, Cache Clearing

Database Maintenance, Search Indexing

Logging and Analytics, PDF Generation

Collect Payments

Note:
* Stock for regular books or special first edition
* Payments for small store or Rolling Stone tickets
* Push notifications for heart surgery or supermarket offer 

---

# say.hi
<!-- .slide: data-background-gradient="linear-gradient(to bottom, #283b95, #17b2c3)" -->
Social Network <!-- .element: class="fragment" -->

### Broadcast messages to all users from a public form  <!-- .element: class="fragment" style="color:red" -->


---

## Simple code

<pre><code data-line-numbers>
public function submitForm(array &$form, FormStateInterface $form_state) {
  $message = $form_state->getValue('message');
  $name = $form_state->getValue('name');

  $users = $this->greetings
    ->sendGreetingsMultiple($name, $message);
}
</code></pre>
---

## Initial diagram

<!-- .slide: data-auto-animate" -->
![alt text](examples/assets/image1.png)

Note:
* This is something we all are used to see. Nothing new
* We need to focus on what happens during the build request phase
* Optimize not only time, but also server resources to increase bandwidth
* Not all the problems are solved increasing memory or adding new servers in parallel, that can be expensive
--

## Zoom into tasks

<!-- .slide: data-auto-animate" -->
![alt text](examples/assets/image2.png)

Note:
* Let's dive into the internals of response generations process
* Multiple tasks, like loading entities, processing entities and prepare rendering
* Task 2 could be a bit problematic, like notifying users or updating related entities when a specific event takes place
* Our site only has a few visitors or nodes and everything is OK

--

## Tasks are getting bigger

<!-- .slide: data-auto-animate" -->
![alt text](examples/assets/image2.png)

Note:
* Our website is more and more popular, we did a great job and it's getting traction
* The number of concurrent users has grown and the number of interaction is bigger than expected
* We get a timeout or out of memory error in our logs from time to time
* Let's increase memory and let's see

--

## Things are broken. It does not scale

<!-- .slide: data-auto-animate" -->
![alt text](examples/assets/image2.png)

Note:
* This is something we all are used to see. Nothing new
* Our site succumbed to success
* Situation does not scale
* We need to improve the Task 2 performance somehow

---

## Does Drupal provide tools to solve this bottleneck?

Note:
* We need to detect where are the bottlenecks. Apparently Task 2
* Improve the code, reduce redundancy, apply caches, etc
* What if we delegates part of these subtasks that are not critical for the main flow?
* Emails, Push notification, image or processing, data import/export, database updates,
  external services synchronization, etc

---

## Batch API
## Cron
## Queues

Note:
* Drupal provides some mechanisms to mitigate these situations
* They're different and cover different use cases
 
---

## Batch API
<!-- .slide: data-auto-animate" -->
![alt text](assets/batch.png)

Note:
* This is screen should be familiar
--

## Batch API
<!-- .slide: data-auto-animate" -->
Tasks are not executed in background

Takes control of the user browser <!-- .element: class="fragment" -->

Maintain resources under control  <!-- .element: class="fragment" -->

**The operation's completion is not certain** <!-- .element: class="fragment" style="color:red" -->

Note:
* This is not a background task, but allows to have resources under control
  for tasks that needs to be executed in the main thread
* Update node permissions table, run database updates or bulk operations
* It can be used also to handle expensive tasks via Drush
* Provided by default when implementing hook_update_N and hook_post_update_NAME

--

## Batch diagram

Note:
* It splits the subtasks to execute them smaller chunks to ensure that server resources are under control
* Logic to determine chunk size and how to manage the resources is up to the developer
* Not very friendly in general. Even more for non-administrative tasks.

--

## Batch Example implementation
<pre><code data-line-numbers="5-13|15-20|22|28-30|35-38|43-52">
public function submitForm(array &$form, FormStateInterface $form_state) {
  ....
  // Set up batch operations.
  $batch = [
    'operations' => [
      ['::initBatch', [count($users)]],
    ],
    'finished' => '::finishedCallback',
    'title' => $this->t('Sending greetings...'),
    'init_message' => $this->t('Starting processing'),
    'error_message' => $this->t('An error occurred during processing'),
  ];

  foreach ($chunks as $chunk) {
    $batch['operations'][] = [
      '\Drupal\say_hi\Form\HiBatchForm::processItems',
      [$chunk, $name, $message],
    ];
  }

  batch_set($batch);
}

/**
 * Inits the batch process.
 */
public static function initBatch(int $total, array &$context): void {
  $context['results']['total'] = $total;
}

/**
 * Sends the greetings.
 */
public static function processItems(array $uids, string $name, string $message, array &$context): void {
  \Drupal::service('say_hi.greetings')
    ->sendGreetingsMultiple($name, $message, $uids);
}

/**
 * Finish batch.
 */
public static function finishedCallback(bool $success, array $results, array $operations): void {
  if ($success) {
    \Drupal::messenger()
      ->addStatus(t('Greetings sent to @count folks.', [
    '@count' => $results['total'],
    ]));
    }
  else {
    \Drupal::messenger()->addError(t('Batch process failed.'));
  }
}
</code></pre>
Note:
* Show Me The Code

---

## Drupal Cron

Based on hook_cron()

Allow to execute recurring or scheduled tasks when Drupal cron is invoked

Note:
* Centralized (for now) in hook_cron()
* Allows to schedule repetitive tasks that can be run in the background
* Remove orphan files from the filesystem, delete fields
* Search API indexation

--

## Limitations

Not all the cron tasks might have same needs

Hard to have a clear overview of tasks run

Overlap / Timeout issues

**[Replace hook_cron() with a more modern approach](https://www.drupal.org/project/drupal/issues/3383487)** <!-- .element: class="fragment" -->

Note:
* Not very flexible
* Custom logic to manage different times
* Need to be careful if tasks take longer than expected (Overlap, Pantheon 5 minutes)

--

## hook_cron()
<!-- .slide: data-auto-animate" -->
<pre data-id="code-animation"><code data-line-numbers>
function my_module_cron() {
  $service = \Drupal('my_module.maintenance'); 
  $service->maintainSite();
}
</code></pre>

Note:
* Show Me The Code

--

## hook_cron()
<!-- .slide: data-auto-animate" -->
<pre data-id="code-animation"><code data-line-numbers="3-5|6-9">
function my_module_cron() {
  $service = \Drupal('my_module.maintenance'); 
  $service->maintainSite();

  $hour = date('H');
  if ($hour >= 8 && $hour <= 10) {
    $service->sayGoodMorning():
  }
}
</code></pre>

Note:
* Show Me The Code

--

## Contrib modules

Simple Cron <!-- .element: class="fragment fade-in-then-semi-out" -->

Ultimate Cron <!-- .element: class="fragment fade-in-then-semi-out" -->

Note:
* Ultimate Cron
* Simple Cron

--

## Custom cron based approaches
<pre><code data-line-numbers="2-3|5-6|8-10|12-13">
# Run the drupal cron process every hour of every day
0 * * * * /usr/bin/wget -O - -q http://mysite/cron/OpgO

# Re-generate the "categories" list (four times a day)
5 0,4,10,16 * * * drush my_module:regenerate-categories

# Rotate the ad banners every 20 minutes
0,20,40  * * * * drush php:eval 
"\Drupal::service('banner_manager')->rotateBanner();"

# Process the mail queue every 2 minutes
*/2 * * * * drush queue:run mail_queue

</code></pre>
Note:
* implement custom 

---

## Queues

Offloads resource-intensive or time-consuming tasks from the main application for improved performance.

Ensures efficient processing by allowing background execution of tasks.

Note:
* Drupal provides Queue API but this is not new
* Lots of different queue providers and most of frameworks provides queue support
* By default, it uses queue table, but new tables can be created as services,
  and it is possible to define new backends
* Generate Media thumbnails
 
--

## Queues

Offloads resource-intensive or time-consuming tasks from the main application for improved performance.

Ensures efficient processing by allowing background execution of tasks.

Note:
* List of items
* create, claim, delete, release
* QueueInterface & ReliableQueueInterface
 
--

## Dealing with queues
<pre><code data-line-numbers="2-3|5-6|10-11|13-20">
// Get the queue.
$my_queue = \Drupal::queue('my_queue');

// Add item to the queue
$my_queue->createItem($my_data);

...

// Claim item from the queue
$item = $my_queue->claimItem();

try {
  $this->process($item->data);
  // Delete item once processed.
  $my_queue->deleteItem($item);
} catch (\Exception $e) {
  // Release item to process it later if failed.
  $queue->releaseItem($item);
}
</code></pre>

Note:
* Create and process a Queue in Drupal is as simple as this
* It can be complex to know when and how to process these queues
* Most the times we just need to iterate over the queue during cron
* Drupal provides QueueWorkers to automate this

--

## QueueWorkers

Automates the generic queue operations to avoid reinventing the wheel

Adds some extra helpful functionalities to manage queues

Note:
* Plugins that define how to process queue items
* It automates the process of claiming items during cron runs and takes care of timeouts
  when queue size is big
* Provides specific exceptions to handle the items in different ways
* Drush & related modules integration

--

## QueueWorker Example
<pre><code data-line-numbers="7-11|19-30|32-34">
namespace Drupal\media\Plugin\QueueWorker;

/**
 * Process a queue of of custom items.
 *
 * @QueueWorker(
 *   id = "media_entity_thumbnail",
 *   title = @Translation("Thumbnail downloader"),
 *   cron = {"time" = 60}
 * )
 */
class ThumbnailDownloader extends QueueWorkerBase {

  /**
   * {@inheritdoc}
   */
  public function processItem(MyData $data) {
    if ($data->delayItem()) {
      // Add item back to the queue but delay execution.
      throw new DelayedRequeueException();
    }
    if ($data->requeueItem()) {
      // Add item back to the queue immediately.
      throw new RequeueException();
    }
    if ($data->suspendQueue()) {
      // Release the item and stop processing the queue.
      throw new SuspendQueueException();
    }

    // If nothing happens the item is deleted
    // from the queue after processing.
    $this->processSuccessful($data);
  }

}
</code></pre>
 
Note:
* Show Me The Code

--

## Contrib modules
External queues: RabbitMQ, Kafka, SQS... <!-- .element: class="fragment fade-in-then-semi-out" -->

Queue Unique  <!-- .element: class="fragment fade-in-then-semi-out" -->

Warmer  <!-- .element: class="fragment fade-in-then-semi-out" -->

Note:
* Queue UI, could not be necessary if Ultimate Cron or Simple Cron are present
* External Queues, like RabbitMQ, Kafka, SQS
* Queue Unique, Warmer

--

## Never Ending Queues

<pre><code class="console">
$ drush queue:list
 ------------------ ------- --------------------------------- 
  Queue              Items   Class                            
 ------------------ ------- --------------------------------- 
  say_hi_greetings   64478   Drupal\Core\Queue\DatabaseQueue  
 ------------------ ------- --------------------------------- 

$ drush queue:run say_hi_greetings
PHP Fatal error:  Allowed memory size of 134217728 bytes...

</code></pre>

Note:
* Queue grows faster than it is processed, so expected tasks are not being executed
* Number of queues overlaps one execution with the next
* Could be mitigated by using Queue Unique module
* Try different cron setups using cron related modules or running queues directly from crontab 

---

## Benchmark

Note:
* Let's get some numbers from an example

--

## Definition

Note:
* Simple module that does a simple thing, theoretically trivial
* Offers a form that allows to send a broadcast email message to all the users
  in the system
* We are offering 3 different forms to perform the same task: main thread, batch and queue

--

## Graphics

Note:
* Show some insights
* Show demo if possible??

--

## Conclusion

Note:
* Show some insights
* Show demo if possible??

---

## When

Note:
* This is probably the biggest question
* Usage of these tools might require some knowledge and requires time to implement
* Need to be realistic. We can leave them out of the MVP, but develop our code having
  in mind that it could be moved into a batch or queue in the future (implement different services) 

---

## Thank you
