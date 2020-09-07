<h1 align="center">
  <img width="275" alt="Adonis Bull" src="https://user-images.githubusercontent.com/16545335/71373449-f2146880-2595-11ea-8a8c-9f51384a3f22.png">
</h1>

<p align="center">
  A <a href="https://github.com/OptimalBits/bull/">Bull</a> provider for the Adonis framework. </br>
  Adonis Bull provides an easy way to start using Bull. The fastest, most reliable, Redis-based queue for Node.
</p>

## Install

You need [install Redis Provider](https://preview.adonisjs.com/guides/database/redis/#setup) before install this package.

YARN

```
yarn add @rocketseat/adonis-bull
```

NPM

```
npm install @rocketseat/adonis-bull
```

## Use

Create a file with the `jobs` that will be processed at `start/jobs.ts`:

```ts
const jobs = ["App/Jobs/UserRegisterEmail"]

export default jobs
```

In the above file you can define redis connections, there you can pass all `Bull` queue configurations described [here](https://github.com/OptimalBits/bull/blob/develop/REFERENCE.md#queue).

Create a new preload file by executing the following ace command.

```bash
node ace make:prldfile bull

# ✔  create    start/bull.ts
```

```js
import Bull from '@ioc:Rocketseat/Bull'

Bull.process()
  // Optionally you can start BullBoard:
  .ui(9999); // http://localhost:9999
  // You don't need to specify the port, the default number is 9999
```

## Creating your job

Create a new job file by executing the following ace command.

```bash
node ace make:job userRegisterEmail

# ✔  create    app/Jobs/UserRegisterEmail.ts
```

```ts
import { JobContract } from '@ioc:Rocketseat/Bull'
import Mail from '@ioc:Adonis/Addons/Mail'

export default class UserRegisterEmail implements JobContract {
  public key = 'UserRegisterEmail'

  public async handle(job) {
    const { data } = job; // the 'data' variable has user data

    await Mail.send("emails.welcome", data, message => {
      message
        .to(data.email)
        .from("<from-email>")
        .subject("Welcome to yardstick");
    });

    return data;
  }
}
```

You can override the default `configs`.

```ts
...
import { JobsOptions, QueueOptions, WorkerOptions, Job } from 'bullmq'

export default class UserRegisterEmail implements JobContract {
  ...
  public options: JobsOptions = {}

  public queueOptions: QueueOptions = {}

  public workerOptions: WorkerOptions = {}
}
```

### Events

You can config the events related to the `job` to have more control over it

```ts
...
import Ws from 'App/Services/Ws'

export default class UserRegisterEmail implements JobContract {
  ...

  boot(queue) {
    queue.on('complete', (job, result) => {
      Ws
        .getChannel('admin:notifications')
        .topic('admin:notifications')
        .broadcast('new:user', result)
    })
  }
}
```

## Processing the jobs

### Simple job

You can share the `job` of any `controller`, `hook` or any other place you might like:

```ts
import User from 'App/Models/User'
import Bull from '@ioc:Rocketseat/Bull'
import Job from 'App/Jobs/UserRegisterEmail'

export default class UserController {
  store ({ request, response }) {
    const data = request.only(['email', 'name', 'password'])

    const user = await User.create(data)


    Bull.add(Job.key, user)
  }
}
```

### Scheduled job

Sometimes it is necessary to schedule a job instead of shooting it imediately. You should use `schedule` for that:

```ts
import User from 'App/Models/User'
import ProductOnSale from 'App/Services/ProductOnSale'
import Bull from '@ioc:Rocketseat/Bull'
import Job from 'App/Jobs/UserRegisterEmail'
import parseISO from 'date-fns/parseISO'

export default class HolidayOnSaleController {
  store ({ request, response }) {
    const data = request.only(['date', 'product_list']) // 2020-11-06T12:00:00

    const products = await ProductOnSale.create(data)

    Bull.schedule(Job.key, products, parseISO(data.date))
  }
}
```

This `job` will be sent only on the specific date, wich for example here is on November 15th at noon.

When finishing a date, never use past dates because it will cause an error.

other ways of using `schedule`:

```ts
Bull.schedule(key, data, new Date("2019-11-15 12:00:00"));
Bull.schedule(key, data, 60 * 1000); // 1 minute from now.
```

Or with a third party lib:

```ts
import humanInterval from 'human-interval'

Bull.schedule(key, data, humanInterval("2 hours")); // 2 hours from now
```

### Advanced jobs

You can use the own `Bull` configs to improve your job:

```ts
Bull.add(key, data, {
  repeat: {
    cron: "0 30 12 * * WED,FRI"
  }
});
```

This `job` will be run at 12:30 PM, only on wednesdays and fridays.

### Exceptions

To have a bigger control over errors that might occur on the line, the events that fail can be manipulated at the file `app/Exceptions/BullHandler.ts`:

```js
import Sentry from 'App/Services/Sentry'

export default class BullHandler {
  async report(error, job) {
    Sentry.configureScope(scope => {
      scope.setExtra(job);
    });

    Sentry.captureException(error);
  }
}
```
