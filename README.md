## About

Timely is useful for two main purposes:

1. A clojure DSL for easier definition of cron strings
2. A scheduling library using cron4j to execute a function according to a scheduled timetable.

## Schedule DSL and Cron

Schedules are a structured way to represent cron syntax and are created using a DSL which reads much like an English sentence.  To get a cron string from a schedule, use schedule-to-cron.  For example:

	timely.core> (schedule-to-cron (each-minute))
	"* * * * *"
	
See the "Define Schedules" section below for more examples of the schedule DSL.

## Define Schedules

Define a scheduled-item using a schedule and a function to be executed on the defined schedule. For example:

````clojure
;; Daily at 12:00am
(scheduled-item (daily)
(test-print-fn 1))
````

(daily) creates a schedule that runs each day at 12:00am.  (test-print-fn 1) returns a function that will print a message.  The combined scheduled-item will print the message each day at 12:00am.

Specific start and end times can be optionally defined to ensure a repeated schedule is only valid for a certain time frame.  This is a feature recognized by the Timely scheduler but does not exist in cron string syntax.

The following are further examples of the dsl for defining schedules:

````clojure
;; Each day at 9:20am
(scheduled-item (daily
                 (at (hour 9) (minute 20)))
                (test-print-fn 2))

;; Each day at 12:00am in april
(scheduled-item (daily
;; [LEO] I like that months can be specified as (month (in-range :april :september)))). I don't like that you can do
;; (month (in-range 4 9)))). Given Java's history with weird offsets in java.util.Calendar
;; (I believe January is indexed to 0, not 1), I would remove the ability to use numbers -- otherwise people who migrated
;; from Java to Clojure are likely to make silly errors that are hard to find.
                 (on (month 4)))
                (test-print-fn 3))

;; Each day at 9:20am in april
(scheduled-item (daily
                 (at (hour 9) (minute 20))
                 (on (month :april)))
                (test-print-fn 4))

;; Monthly on the 3rd at 9:20am
(scheduled-item (monthly
                 (at (hour 9) (minute 20) (day 3)))
                (test-print-fn 5))

;; Between months 4-9 on Mondays, each hour
(scheduled-item (hourly
                 (on (day-of-week :mon)
                     (month (in-range 4 9))))
                (test-print-fn 6))

;; Between months 4-9 on Mondays and Fridays, each hour
(scheduled-item (hourly
                 (on (day-of-week :mon :fri)
;; [LEO] I notice you can do (in-range :april :september). Can you do (in-range :monday :friday) as well?
                     (month (in-range :april :september))))
                (test-print-fn 7))

;; [LEO] On a side note, A reference sheet (in addition to examples) would be great... otherwise one has to
;; look at the code to understand what will work and what will not

;; On every 8am and 5pm
(scheduled-item (daily
;; [LEO] commas might be useful here so that it's immediately clear that you're doing hours 8 and 17, not 8 thru 17.
;; So maybe (at (hour 8, 17)))? Maybe the example throws me off because 8am-5pm sounds like regular working hours,
;; and I'm used to seeing those in a range instead of as a pair.
                 (at (hour 8 17)))
                (test-print-fn 8))

;; [LEO] Any thoughts on supporting am and pm? Something like (at (hour (am 8) (pm 5))? I don't know if that's useful or tacky.

;; On monday and wednesday at 9:20am
(scheduled-item (on-days-of-week
                 [:mon :wed]
                 (at (hour 9) (minute 20)))
                (test-print-fn 9))

;; Every 2 minutes
(scheduled-item (every :minute
                       2)
                (test-print-fn 10))

;; Every 2 minutes, but only every 2 days
(scheduled-item (every :minute
                       2
;; [LEO] This is the syntax for "every 2 days"? I find it confusing. It looks much more like "twice per day" than "every 2 days"
                       (per :day 2))
                (test-print-fn 11))

;; Every 2:01am on April 3rd
(scheduled-item (create-schedule
                 1 2 3 4 all)
                (test-print-fn 12))

;; Start time in the future
(scheduled-item (each-minute
;; [LEO] I think the syntax here loses some of it's magic. Up to know, everything looks like English
;; (:every 2 minutes, (at (hour 9)), etc), but now you are doing raw data/time manipulation. Can similar syntax be
;; used for start times? Something like:
;;    (starting (at (month :june) (day 1))) ;; start on june 1st
;;             or
;;    (starting (in (hours 9) (minutes 5))) ;; start 9:05 from now
                 (start-time (*  (dates-coerce/to-long (dates/now)) 2)))
                (test-print-fn 13))

;; [LEO] what's the purpose of this example?
;; End time already passed
(scheduled-item (each-minute
                 (end-time 0))
                (test-print-fn 14))

;; Is within range
(scheduled-item (each-minute
;; [LEO] how about a shortcut for this use case? Instead of (start-time 0) maybe (start-immediately) (which calls (start-time 0) under the hood)
                 (start-time 0)
;; [LEO] also, a shortcut for (run-forever) might be nice)
                 (end-time (* (dates-coerce/to-long (dates/now)) 2)))
                (test-print-fn 15))

;; Schedule within a specific time range
(scheduled-item
 (each-minute
  (start-time (to-utc-timestamp (dates/date-time 2012 5 15 11 42)))
  (end-time (to-utc-timestamp (dates/date-time 2012 5 15 11 43))))
 (test-print-fn "specific-time-range"))
````     
          
## Run Schedules

Use (start-scheduler) to enable scheduling in your application.

Use start-schedule and end-schedule to start and stop schedules in your application:

````clojure
(start-scheduler)
(let [item (scheduled-item
            (each-minute)
            (test-print-fn "Scheduled using start-schedule"))]
  (let [sched-id (start-schedule item)]
    (Thread/sleep (* 1000 60 2))
    (end-schedule sched-id)))
````
