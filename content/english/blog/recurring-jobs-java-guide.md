---
title: "Recurring Jobs in Java: Schedules That Actually Work"
description: "How to implement reliable recurring jobs in Java. Cron expressions, missed job handling, timezone support, and distributed scheduling that doesn't run twice."
image: /blog/recurring-jobs-java.webp
date: 2026-02-03T13:00:00+02:00
author: "Nicholas D'hondt"
tags:
  - blog
  - recurring jobs
  - scheduling
  - tutorial
---

You need a job to run every day at 9am. Or every hour. Or every Monday. Simple requirement, right?

Then you deploy to production and discover:
- The job runs twice because you have two servers
- It skips runs when the app restarts during scheduled time
- Daylight saving time breaks your schedule
- You have no idea if it actually ran

Recurring jobs are simple in theory, complex in practice. Let's fix that.

## The Problem With Simple Solutions

### ScheduledExecutorService

Java's built in option:

```java
ScheduledExecutorService executor = Executors.newScheduledThreadPool(1);
executor.scheduleAtFixedRate(
    () -> generateDailyReport(),
    0,
    24,
    TimeUnit.HOURS
);
```

**Problems:**
- No cron expressions (what about "every weekday at 9am"?)
- Lost on restart (your report doesn't run if app restarts at 8:59am)
- Drifts over time (fixed rate doesn't account for execution time)
- No visibility (did it run? did it fail?)

### Spring @Scheduled

Better, but still limited:

```java
@Scheduled(cron = "0 0 9 * * *")
public void dailyReport() {
    reportService.generate();
}
```

**Problems:**
- Multiple instances = multiple executions (two servers = two reports)
- No persistence (restart during scheduled time = missed run)
- No retry on failure (exception = silent failure until tomorrow)
- No dashboard (you find out it's broken when someone asks where the report is)

### Timer + TimerTask

Don't. Just don't. It's 2026.

## What You Actually Need

A production recurring job system handles:

1. **Distributed locking.** One execution across your cluster.
2. **Persistence.** Jobs survive restarts.
3. **Missed run recovery.** If the app was down at 9am, run at 9:15am when it comes back.
4. **Timezone support.** 9am means 9am in your timezone, not UTC.
5. **Monitoring.** See what ran, what failed, when it runs next.
6. **Cron expressions.** Real scheduling flexibility.

## JobRunr: Recurring Jobs That Work

Here's how to set up a recurring job that handles all of the above:

```java
BackgroundJob.scheduleRecurrently(
    "daily-report",
    Cron.daily(9, 0),
    ZoneId.of("Europe/Brussels"),
    () -> reportService.generateDailyReport()
);
```

That's it. Let's break down what this gives you.

### Distributed by Default

Run this code on 10 servers. The job runs once. JobRunr uses your database for coordination. Optimistic locking ensures exactly one execution.

No additional configuration. No separate locking service. Just works.

### Survives Restarts

The schedule is stored in your database. Restart your app, redeploy, scale up, scale down. The recurring job keeps running on schedule.

When your app starts, JobRunr reads the schedule from the database and picks up where it left off.

### Handles Missed Runs

App was down during scheduled time? By default, JobRunr runs the job when the app comes back up.

Want different behavior? Configure it:

```java
// Skip missed runs (only run at scheduled time)
BackgroundJob.scheduleRecurrently(
    RecurringJobId.create("daily-report"),
    "daily-report",
    Cron.daily(9, 0),
    () -> reportService.generateDailyReport()
);
```

### Timezone Aware

Specify the timezone and JobRunr handles daylight saving transitions:

```java
BackgroundJob.scheduleRecurrently(
    "daily-report",
    Cron.daily(9, 0),
    ZoneId.of("America/New_York"),
    () -> reportService.generateDailyReport()
);
```

9am in New York means 9am in New York, whether it's EST or EDT.

### Built-in Dashboard

Open `/dashboard` and see:

- All recurring jobs and their schedules
- Next run time for each job
- History of past executions
- Failed runs with exception details
- Manual trigger button for testing

<figure>
<img src="/documentation/recurring-jobs-702x228.webp" class="kg-image">
<figcaption>JobRunr dashboard showing recurring jobs</figcaption>
</figure>

## Common Recurring Job Patterns

### Daily at Specific Time

```java
// Every day at 9:00 AM
BackgroundJob.scheduleRecurrently(
    "morning-report",
    Cron.daily(9, 0),
    () -> reportService.morningReport()
);

// Every day at 11:30 PM
BackgroundJob.scheduleRecurrently(
    "nightly-cleanup",
    Cron.daily(23, 30),
    () -> cleanupService.nightlyCleanup()
);
```

### Multiple Times Per Day

```java
// Every 4 hours
BackgroundJob.scheduleRecurrently(
    "sync-inventory",
    "0 */4 * * *",
    () -> inventoryService.sync()
);

// At 9am and 5pm
BackgroundJob.scheduleRecurrently(
    "status-check",
    "0 9,17 * * *",
    () -> statusService.check()
);
```

### Weekdays Only

```java
// Weekdays at 9am
BackgroundJob.scheduleRecurrently(
    "weekday-standup",
    "0 9 * * 1-5",
    () -> slackService.postStandupReminder()
);

// Business hours every 30 minutes
BackgroundJob.scheduleRecurrently(
    "business-hours-check",
    "*/30 9-17 * * 1-5",
    () -> alertService.checkSystems()
);
```

### Weekly

```java
// Every Monday at 9am
BackgroundJob.scheduleRecurrently(
    "weekly-report",
    Cron.weekly(DayOfWeek.MONDAY, 9),
    () -> reportService.weeklyReport()
);

// Every Sunday at midnight
BackgroundJob.scheduleRecurrently(
    "weekly-cleanup",
    "0 0 * * 0",
    () -> cleanupService.weeklyCleanup()
);
```

### Monthly

```java
// First day of month at midnight
BackgroundJob.scheduleRecurrently(
    "monthly-invoice",
    Cron.monthly(1),
    () -> billingService.generateInvoices()
);

// 15th of every month at 9am
BackgroundJob.scheduleRecurrently(
    "mid-month-review",
    "0 9 15 * *",
    () -> reviewService.midMonthReview()
);
```

### Yearly

```java
// January 1st at midnight
BackgroundJob.scheduleRecurrently(
    "yearly-archive",
    "0 0 1 1 *",
    () -> archiveService.archivePreviousYear()
);
```

## Updating Recurring Jobs

Need to change a schedule? Just call `scheduleRecurrently` again with the same ID:

```java
// Original: daily at 9am
BackgroundJob.scheduleRecurrently(
    "daily-report",
    Cron.daily(9, 0),
    () -> reportService.generateDailyReport()
);

// Changed to: daily at 10am
BackgroundJob.scheduleRecurrently(
    "daily-report",
    Cron.daily(10, 0),
    () -> reportService.generateDailyReport()
);
```

The schedule updates automatically. No need to delete and recreate.

## Deleting Recurring Jobs

```java
BackgroundJob.deleteRecurringJob("daily-report");
```

Or use the dashboard to delete jobs manually.

## Handling Failures

Recurring jobs automatically retry on failure with exponential backoff:

1. First retry: ~15 seconds later
2. Second retry: ~30 seconds later
3. Third retry: ~1 minute later
4. ...and so on

After 10 retries, the job moves to the failed state. You'll see it in the dashboard with the full exception.

Want different retry behavior? Configure it:

```java
@Job(name = "Daily Report", retries = 5)
public void generateDailyReport() {
    // Your logic
}
```

## Long Running Recurring Jobs

If your recurring job takes longer than the interval, JobRunr handles it gracefully. The next run starts after the current one finishes, not on top of it.

Example: Job scheduled every hour, takes 90 minutes to run.
- 9:00 AM: Job starts
- 10:00 AM: Scheduled run skipped (previous still running)
- 10:30 AM: Previous job finishes
- 11:00 AM: Next job starts

No overlapping executions. No pile-up of waiting jobs.

## Testing Recurring Jobs

### Trigger Manually

From the dashboard, click "Trigger" on any recurring job to run it immediately. Useful for testing without waiting for the schedule.

### In Tests

```java
@Test
void testRecurringJob() {
    // Register the recurring job
    BackgroundJob.scheduleRecurrently(
        "test-job",
        Cron.daily(9, 0),
        () -> myService.doWork()
    );
    
    // Trigger it manually
    jobScheduler.triggerRecurringJob("test-job");
    
    // Wait and verify
    await().atMost(Duration.ofSeconds(10))
        .untilAsserted(() -> verify(myService).doWork());
}
```

## Migration from @Scheduled

If you're moving from Spring @Scheduled:

**Before:**
```java
@Component
public class ScheduledTasks {
    
    @Scheduled(cron = "0 0 9 * * *")
    public void dailyReport() {
        reportService.generate();
    }
    
    @Scheduled(cron = "0 */30 * * * *")
    public void syncData() {
        syncService.sync();
    }
}
```

**After:**
```java
@Component
public class RecurringJobsConfig {
    
    private final ReportService reportService;
    private final SyncService syncService;
    
    @PostConstruct
    public void registerRecurringJobs() {
        BackgroundJob.scheduleRecurrently(
            "daily-report",
            Cron.daily(9, 0),
            reportService::generate
        );
        
        BackgroundJob.scheduleRecurrently(
            "sync-data",
            "*/30 * * * *",
            syncService::sync
        );
    }
}
```

**What you gain:**
- Jobs survive restarts
- No duplicate execution across instances
- Dashboard visibility
- Automatic retries
- History of past runs

## Setup

Add JobRunr to your project:

```xml
<dependency>
    <groupId>org.jobrunr</groupId>
    <artifactId>jobrunr-spring-boot-3-starter</artifactId>
    <version>7.3.2</version>
</dependency>
```

Configure in `application.yml`:

```yaml
org:
  jobrunr:
    background-job-server:
      enabled: true
    dashboard:
      enabled: true
```

Register your recurring jobs at startup and you're done.

---

*Need more than recurring jobs? JobRunr also handles [fire-and-forget jobs](/en/documentation/background-methods/enqueueing-jobs/), [delayed jobs](/en/documentation/background-methods/scheduling-jobs/), and [job workflows](/en/documentation/pro/job-chaining/).*

*Build cron expressions visually with our [Cron Expression Generator](/en/tools/cron-expression-generator/).*
