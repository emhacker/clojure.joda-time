# Clojure.Joda-Time

An idiomatic Clojure wrapper for Joda-Time.

Main goals:

* Provide a consistent API for common operations with
  instants, date-times, periods, partials and intervals.
* Provide an escape hatch from Joda types to clojure datastructures
  and back where possible.
* Avoid reflective calls (this is a problem, because many types in Joda-Time
  have similar functionality hidden under similarly named and overloaded
  methods with no common interfaces).
* Provide an entry point into Joda-Time by freeing the user from importing most
  of the Joda-Time classes.

Compared to [clj-time](https://github.com/clj-time/clj-time), this library is
not `DateTime`-centric. If you tend to use **local** dates in most of your
projects, meaning you don't care about the time zones, there's no purpose in
using `DateTime` at all. You should be using various `Partials` provided by
Joda-Time, most common being `LocalDate` and `LocalDateTime`. This also means
that date-times created through Clojure.Joda-Time are not converted to the UTC
timezone by default, as they are in **clj-time**.

## Usage

API of the Clojure.Joda-Time consists of one namespace, namely: `joda-time`.
For the purposes of this guide, we will `use` the main namespace:

    (refer-clojure :exclude (merge partial iterate print))
    (use 'joda-time)

### An appetizer

First, a quick run through common use cases.

What is the current date and time in our time zone?

    (def now (date-time))
    => #<DateTime 2013-12-10T13:07:16.000+02:00>

In UTC?

    (with-zone now (timezone :UTC))
    => #<DateTime 2013-12-10T11:07:16.000Z>

Without the time zone?

    (def now-local (local-date-time))
    => #<LocalDateTime 2013-12-10T13:07:16.000>

Now, how would we go about a date five years and six months from now?  First,
we would need to represent this period:

    (period {:years 5, :months 6})
    => #<Period P5Y6M>

or as a sum:

    (def five-years-and-some (plus (years 5) (months 6)))
    => #<Period P5Y6M>

Now for the date:

    (def in-five-years (plus now five-years-and-some))
    => #<DateTime 2019-06-10T13:07:16.000+03:00>

    (def in-five-years-local (plus now-local (period {:years 5, :months 6})))
    => #<LocalDateTime 2019-06-10T13:07:16.000>

How many hours to the point five years and six months from now?

    (hours-in now in-five-years)
    => 48191

    (hours-in now-local in-five-years-local)
    => 48191

What if we want a specific date?

    (def in-two-years (date-time "2015-12-10"))
    => #<DateTime 2015-12-10T00:00:00.000+02:00>

    (def in-two-years-local (local-date-time "2015-12-10"))
    => #<LocalDateTime 2015-12-10T00:00:00.000>

Does the interval from `now` to `in-five-years` contain this date?

    (after? in-five-years now in-two-years)
    => true

    (after? in-five-years-local now-local in-two-years-local)
    => true

Another way, actually using the interval type:

    (contains? (interval now in-five-years) in-two-years)
    => true

    (contains? (partial-interval now-local in-five-years-local) in-two-years-local)
    => true

What about the current day of month?

    (-> now (property :dayOfMonth) value)
    => 10

    (-> now-local (property :dayOfMonth) value)
    => 10

The date at the last day of month?

    (-> now (property :dayOfMonth) with-max-value)
    => #<DateTime 2013-12-31T13:07:16.000+02:00>

    (def new-years-eve (-> now-local (property :dayOfMonth) with-max-value)
    => #<LocalDateTime 2013-12-31T13:07:16.000>

Every date at the last day of month from now?

    (iterate plus new-years-eve (months 1))
    => (#<LocalDateTime 2013-12-31T13:07:16.000>
        #<LocalDateTime 2014-01-31T13:07:16.000> ...)

In case we want to print the dates, we'll need a formatter:

    (def our-formatter (formatter "yyyy/MM/dd"))
    => #<DateTimeFormatter ...>

    (print our-formatter now)
    => "2013/12/10"

    (print our-formatter now-local)
    => "2013/12/10"

And what about parsing?

    (j/parse-date-time our-formatter "2013/12/10")
    => #<DateTime 2013-12-10T00:00:00.000+02:00>

    (j/parse-local-date our-formatter "2013/12/10")
    => #<LocalDate 2013-12-10>

I hope you're interested. However, we've barely scratched the surface of the
API. Please, continue reading for a deeper look.

### Joda-Time entities

Clojure.Joda-Time provides a way to construct most of the time entities
provided by the Joda-Time. For example, given a `LocalDate` type in Joda-Time,
the corresponding construction function (I'll hijack the name "constructor" to
define construction functions) in Clojure.Joda-Time will be called
`local-date`.

A call to a constructor with a single argument goes through the following
pattern:

* given a `nil`, return a `nil` (**important**: this is different from the
  default Joda-Time behaviour which usually has a default value for `nil`)
* given a number, convert it to `Long` and invoke the next rule,
* given a map, try to reconstruct a time entity from its map representation
  (see **Properties** section) or invoke one of the constructors on the
  corresponding Java class.
* given any object, pass it to the Joda-Time `ConverterManager`,

Mostly single-argument constructors are supported (except for a several cases
which we will look at later on) to avoid confusion with overloading.

By convention, a call to a constructor without arguments will return a time
entity constructed at the current date and time.

#### Instants

In Joda-Time [instants](http://www.joda.org/joda-time/key_instant.html) are
represented by `DateTime`, `MutableDateTime` and `Instant` types.

    (date-time)
    => #<DateTime 2013-12-10T10:20:13.133+02:00>

    (mutable-date-time)
    => #<MutableDateTime 2013-12-10T10:20:15.295+02:00>

    (instant)
    => #<Instant 2013-12-10T10:20:16.233+02:00>

You might have noticed that `DateMidnight` is not supported. This is because
the type is deprecated in the recent versions of Joda-Time. If you need a
midnight date, you should be use:

    (.withTimeAtStartOfDay (date-time))
    => #<DateTime 2013-12-10T00:00:00.000+02:00>

Also, there are no overloads for date-time constructors which specify a year, a
year and month, etc. The reason is I don't find them especially useful - you
can always construct a date at a given date by using an ISO-formatted string:

    (date-time "2010")
    => #<DateTime 2010-01-01T00:00:00.000+02:00>

    (date-time "2010-12-20")
    => #<DateTime 2010-12-20T00:00:00.000+02:00>

This has a disadvantage of not being able to partially apply a constructor,
like:

    (def at-year-2010 (partial (date-time 2010)))

For now I'm not convinced that multi-argument date-time constructors should be
supported as in **clj-time** (you're welcome to open a ticket and describe your
reasons).

#### Partials

[Partials](http://www.joda.org/joda-time/key_partial.html) are represented by
`Partial`, `LocalDate`, `LocalDateTime`, `LocalTime`, `YearMonth` and `MonthDay`.

    (partial)
    #<Partial []>

    (partial {:year 2013, :monthOfYear 12})
    #<Partial 2013-12>

    (local-date)
    #<LocalDate 2013-12-10>

    (local-date-time)
    #<LocalDateTime 2013-12-10T10:15:13.553>

    (local-time)
    #<LocalTime 10:15:15.234>

    (year-month)
    #<YearMonth 2013-12>

    (month-day)
    #<MonthDay --12-10>

#### Periods

Joda types for [periods](http://www.joda.org/joda-time/key_period.html) are
`Period`, `MutablePeriod`, `Years`, `Months`, `Weeks`, `Days`, `Hours`,
`Minutes` and `Seconds`.

Multi-field period accepts a map of several possible shapes. The first shape is
a map representation of a period, e.g.:

    (period {:years 10, :months 10})
    => #<Period P10Y10M>

the second shape delegates to an appropriate Joda `Period` constructor, such
as:

    (period {:start 0, :end 1000})
    => #<Period PT1S>

Period constructor can be called with two arguments, where the second argument
is the type of the period - either a `PeriodType` or a vector of duration field
name keywords:

    (period 1000 [:millis])
    => #<Period PT1S>

    (period {:start 0, :end 1000} (period-type :millis))
    => #<Period PT1S>

All of the single-field periods are constructed the same way:

    (years 10)
    => #<Years P10Y>

    (months 5)
    => #<Months P5M>

Single-field period constructors can also be used to extract duration component
out of the multi-field period:

    (years (period {:years 20, :months 10}))
    => #<Years P20Y>

When called on an interval, single-field period constructor will calculate the
duration (same as `Years.yearsIn`, `Months.monthsIn`, etc. in Joda-Time):

    (minutes (interval (date-time "2008") (date-time "2010")))
    => #<Minutes PT1052640M>

You can get the value of the single-field period using the helper functions:

    (minutes-in (period {:years 20, :minutes 10}))
    => 10

    (minutes-in (interval (date-time "2008") (date-time "2010")))
    => 1052640

    (minutes-in (duration (* 1000 1000))
    => 16

You can also query the type of the period:

    (period-type (period))
    => #<PeriodType PeriodType[Standard]>

    (period-type (years 5))
    => #<PeriodType PeriodType[Years]>

`period-type` can also construct a `PeriodType` out of the duration field type
names:

    (period-type :years)
    => #<PeriodType PeriodType[Years]>

    (period-type :years :months :weeks :days)
    => #<PeriodType PeriodType[StandardNoHoursNoMinutesNoSecondsNoMillis]>

You can also convert the `PeriodType` back to a seq of keywords:

    (period-type->seq (period-type (years 10)))
    => [:years]

#### Durations

[Duration](http://www.joda.org/joda-time/key_duration.html) is the most
boring time entity. It can be constructed in the following way:

    (duration 1000)
    => #<Duration PT1S>

Duration constructor also accepts a map (you can find the whole set of options
in the docstring):

    (duration {:start (date-time), :period (years 5)})
    => #<Duration PT157766400S>

#### Intervals

[Intervals](http://www.joda.org/joda-time/key_interval.html) consist of two
types inherited from Joda Time:

    (interval 0 1000)
    => #<Interval 1970-01-01T00:00:00.000Z/1970-01-01T00:00:01.000Z>

    (mutable-interval 0 1000)
    => #<MutableInterval 1970-01-01T00:00:00.000Z/1970-01-01T00:00:01.000Z>

As you can see, the interval constructor accepts start and end arguments -
either milliseconds from epoch, instants or date-times. Constructor also
accepts a map with different combinations of `start`, `end`, `duration` and
`period` parameters, same as Joda-Time `Interval` constructors:

    (interval {:start 0, :end 1000})
    => #<Interval 1970-01-01T00:00:00.000Z/1970-01-01T00:00:01.000Z>

    (interval {:start 0, :duration 1000})
    => #<Interval 1970-01-01T00:00:00.000Z/1970-01-01T00:00:01.000Z>

    (interval {:start 0, :period (seconds 1)})
    => #<Interval 1970-01-01T00:00:00.000Z/1970-01-01T00:00:01.000Z>

There is also an implementation of a Partial interval which consists of two
partials with equal types:

    (partial-interval (partial {:year 0}) (partial {:year 2010}))
    => #joda_time.interval.PartialInterval{:start #<Partial 0000>, :end #<Partial 2010>}

    (partial-interval (local-date "2010") (local-date "2013"))
    => #joda_time.interval.PartialInterval{:start #<LocalDate 2010-01-01>,
                                           :end #<LocalDate 2013-01-01>}

record representation of the partial interval is an implementation detail and
should not be relied upon.

Both instant and partial intervals support a common set of operations on their
start/end (string representation of the interval is shortened for readability):

    (def i (interval 0 10000))
    => #<Interval 00.000/10.000>

    (move-start-to i (instant 5000))
    => #<Interval 05.000/10.000>

    (move-end-to i (instant 5000))
    => #<Interval 00.000/05.000>

    (move-start-by i (seconds 5))
    => #<Interval 05.000/10.000>

    (move-end-by i (seconds 5))
    => #<Interval 05.000/15.000>

    (move-end-by i (seconds 5))
    => #<Interval 05.000/15.000>

intervals can also be queried for several properties:

    (start i)
    => #<DateTime 1970-01-01T00:00:00.000Z>

    (end i)
    => #<DateTime 1970-01-01T00:00:10.000Z>

    (contains? i (interval 2000 5000))
    => true

    (contains? i (interval 0 15000))
    => false

    (contains? (interval (date-time "2010") (date-time "2012"))
               (date-time "2011"))
    => true

    (overlaps? (interval (date-time "2010") (date-time "2012"))
               (interval (date-time "2011") (date-time "2013")))
    => true

    (abuts? (interval (date-time "2010") (date-time "2012"))
            (interval (date-time "2012") (date-time "2013")))
    => true

we can also calculate interval operations present in the Joda-Time:

    (overlap (interval (date-time "2010") (date-time "2012"))
             (interval (date-time "2011") (date-time "2013")))
    => #<Interval 2011/2012>

    (gap (interval (date-time "2010") (date-time "2012"))
         (interval (date-time "2013") (date-time "2015")))
    => #<Interval 2012/2013>

All of the above functions work with partial intervals the same way.

#### Timezones and Chronologies

Timezones can be constructed through the `timezone` function given the
(case-sensitive) timezone ID:

    (timezone)
    => #<CachedDateTimeZone Europe/Vilnius>

    (timezone :UTC)
    => #<FixedDateTimeZone UTC>

Chronologies are constructed using `chronology` with a lower-case chronology
type and an optional timezone argument:

    (chronology :coptic)
    => #<CopticChronology CopticChronology [Europe/Vilnius]>

    (chronology :coptic :UTC)
    => #<CopticChronology CopticChronology [UTC]>

    (chronology :iso (timezone :UTC))
    => #<ISOChronology ISOChronology [UTC]>

#### Formatters

Formatters (printers and parsers) are defined through the `formatter` function:

    (formatter "yyyy-MM-dd")
    => #<DateTimeFormatter ...>

All of the ISO formatter defined by Joda-Time in the `ISODateTimeFormat` class
can be referenced by the appropriate keywords:

    (formatter :date-time)
    => #<DateTimeFormatter ...>

Formatters may also be composed out of multiple patterns and other formatters:

    (def fmt (formatter "yyyy/MM/dd" :date-time (formatter :date)))
    => #<DateTimeFormatter ...>

the resulting formatter will print according to the first pattern:

    (print fmt (date-time "2010"))
    => "2010/01/01"

and parse all of the provided formats. Dates can be parsed from strings using
a family of `parse` functions:

    (parse-date-time fmt "2010/01/01")
    => #<DateTime 2010-01-01T00:00:00.000+02:00>

    (parse-mutable-date-time fmt "2010/01/01")
    => #<MutableDateTime 2010-01-01T00:00:00.000+02:00>

    (parse-local-date fmt "2010/01/01")
    => #<LocalDate 2010-01-01>

    (parse-local-date-time fmt "2010/01/01")
    => #<LocalDateTime 2010-01-01T00:00:00.000>

    (parse-local-time fmt "2010/01/01")
    => #<LocalTime 00:00:00.000>

### Properties

Properties allow us to query and act on separate fields of date-times,
instants, partials and periods.

We can query single properties by using the `property` function:

    (value (property (date-time "2010") :monthOfYear))
    => 1

    (max-value (property (instant "2010") :monthOfYear))
    => 12

    (min-value (property (partial {:monthOfYear 10}) :monthOfYear))
    => 1

    (with-value (property (period {:years 10, :months 5}) :years) 15)
    => #<Period P15Y5M>

Property expressions read better when chained with threading macros:

    (-> (date-time "2010") (property :monthOfYear) value)
    => 1

Clojure loves maps, so I've tried to produce a map interface to the most
commonly used Joda-time entities. Date-times, instants, partials and periods
can be converted into maps using the `properties` function which uses
`property` under the hood.  For example, a `DateTime` contains a whole bunch of
properties - one for every `DateTimeFieldType`:

    (def props (properties (date-time)))
    => {:centuryOfEra #<Property Property[centuryOfEra]>, ...}

    (keys props)
    => (:centuryOfEra :clockhourOfDay :clockhourOfHalfday
        :dayOfMonth :dayOfWeek :dayOfYear
        :era :halfdayOfDay :hourOfDay :hourOfHalfday
        :millisOfDay :millisOfSecond :minuteOfDay
        :minuteOfHour :monthOfYear :secondOfDay :secondOfMinute
        :weekOfWeekyear :weekyear :weekyearOfCentury
        :year :yearOfCentury :yearOfEra)

Now we can get the values for all of the fields (we'll cheat and use
`flatland.useful.map/map-vals`):

    (useful/map-vals props value)
    => {:year 2013, :monthOfYear 12, :dayOfMonth 8,
        :yearOfCentury 13, :hourOfHalfday 9, :minuteOfDay 1305, ...}

although this is better achieved by calling `as-map` convenience function.

As you can see, map representations allow us to plug into the rich set of
operations on maps provided by Clojure and free us from using
`DateTimeFieldType` or `DurationFieldType` classes directly.

Partials contain a smaller set of properties, for example:

    (-> (partial {:year 2013, :monthOfYear 12})
        properties
        (useful/map-vals value))
    => {:year 2013, :monthOfYear 12}

Properties allow us to perform a bunch of useful calculations, such as getting
the date for the last day of the current month:

    (-> (date-time) (property :dayOfMonth) with-max-value)

or get the date for the first day:

    (-> (date-time) (properties :dayOfMonth) with-min-value)

We can also solve a common problem of getting a sequence of dates for the last
day of month:

    (iterate plus
             (-> (local-date) (property :dayOfMonth) with-max-value)
             (months 1))
    => (#<LocalDate 2013-12-31> #<LocalDate 2014-01-31> ...)

Note that `iterate` is defined in the `joda-time` namespace.

### Operations

One of the most useful parts of the Joda-Time library is it's rich set of
arithmetic operations allowed on the various time entities. You can sum periods
and durations together or add them to date-times, instants or partials.  You
can compute the difference of durations and periods or subtract them from
dates. You can also negate and compute absolute values of durations and
periods.

Here's an example of using a `plus` operation on a date-time:

    (def now (date-time "2010-01-01"))
    => #<DateTime 2010-01-01T00:00:00.000+02:00>

    (plus now (years 11))
    => #<DateTime 2021-01-01T00:00:00.000+02:00>

    (def millis-10sec (* 10 1000))
    => 10000

    (def duration-10sec (duration millis-10sec)
    => #<Duration PT10S>

    (plus now (years 11) (months 10) (days 20) duration-10sec millis-10sec)
    => #<DateTime 2021-11-21T00:00:20.000+02:00>

same with instants:

    (plus (instant "2010-01-01") (years 11))
    => #<Instant 2021-01-01T00:00:00.000Z>

with partials:

    (def now (local-date "2010-01-01"))
    => #<LocalDate 2010-01-01>

    (plus now (years 11) (months 10) (days 20))
    => #<LocalDate 2021-11-21>

or with periods:

    (def p (plus (years 10) (years 10) (months 10)))
    => #<Period P20Y10M>

    (period-type p)
    => #<PeriodType PeriodType[StandardNoWeeksNoDaysNoHoursNoMinutesNoSecondsNoMillis]>

or with durations:

    (plus (duration 1000) (duration 1000) 1000)
    => #<Duration PT3S>

Obviously, you can `minus` all the same things you can `plus`:

    (minus now (years 11) (months 10))
    => #<LocalDate 1998-03-01>

    (minus (duration 1000) (duration 1000) 1000)
    => #<Duration PT-1S>

    (minus (years 10) (years 10) (months 10))
    => #<Period P-10M>

As you can see, durations and periods can become negative. Actually, we can
turn a positive period into a negative one by using `negate`:

    (negate (years 10))
    => #<Years P-10Y>

    (negate (duration 1000))
    => #<Duration PT-1S>

and we can take an absolute value of a period or a duration:

    (abs (days -20))
    => #<Days P20D>

    (abs (days 20))
    => #<Days P20D>

    (abs (duration -1000))
    => #<Duration PT1S>

There is also a `merge` operation which is supported by periods and partials.
In case of a period, `merge` works like `plus`, only the values get overwritten
like when merging maps with `clojure.core/merge`:

    (merge (period {:years 10, :months 6}) (years 20) (days 10))
    => #<Period P20Y6M10D>

    (merge (local-date) (local-time))
    => #<Partial 2013-12-09T11:10:00.350>

    (merge (local-date) (local-time) (partial {:era 0})
    => #<Partial [era=0, year=2013, monthOfYear=12, dayOfMonth=9,
                  hourOfDay=11, minuteOfHour=10, secondOfMinute=5,
                  millisOfSecond=429]>

Essentially, merging several partials or periods together is the same as
converting them to their map representations with `as-map`, merging maps and
converting the result back into a period/partial, only in a more efficient way.

It's important to note that operations on mutable Joda-Time entities aren't
supported. You are expected to chain methods through java interop.

## License

Copyright © 2013 Vadim Platonov

Distributed under the MIT License.