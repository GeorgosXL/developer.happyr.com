---
title: Testing the new symfony router
author: Georgos Landin Xanthopoulos
date: '2018-02-02 15:06:47 +0200'
header:
  teaser: images/posts/teaser2.jpg
  caption: "Photo credit: [**The Preiser Project**](https://www.flickr.com/photos/thepreiserproject/)"
categories:
- Symfony
---
<b>How much faster is the new Symfony router in a real world application?</b>

<b>We ran some tests to find out.</b>

## An improved router

A few days ago we received the fantastic news that the Symfony router had been improved significantly, of course we wanted to see how
the changes would affect our application, so we decided to test it against our own routes. 

If you somehow missed the exciting news you can read the official announcement here: 
<https://symfony.com/blog/new-in-symfony-4-1-fastest-php-router>

If you want to know how the new router works, read Nicolas Grekas [post](https://medium.com/@nicolas.grekas/making-symfonys-router-77-7x-faster-1-2-958e3754f0e1) where he dives more into detail about the new Symfony router. 

## Our setup 
* PHP version: 7.1.13
* Static routes: 285
* Dynamic routes: 350

## The test process 
The process in it self is quite simple. Since the performance of the matcher is based on the amount of configured routes that we have, 
we want to test against the first route, five random routes, and the last route, to get a realistic test case. 
This was done for both the static and the dynamic routes. 

{% highlight php%}
class RouteTestCommand extends BaseCommand
{
    const NR_OF_MATCHES = 50000;

    protected static $defaultName = 'route:test';

    /** @var Router */
    private $router;

    public function __construct(RouterInterface $router)
    {
        parent::__construct();
        $this->router = $router;
    }
    
    protected function execute(InputInterface $input, OutputInterface $output)
    {
        $output->writeln('First static route: '.$this->matchRoutes(['/admin/statistics/index']));        
        // Add random/last static routes..

        $output->writeln('First dynamic route: '.$this->matchRoutes(['/admin/statistics/3aa63971-1f61-4b28-bd60-5bb1c9582c8a/show']));
        // Add random/last dynamic routes..
        
        $output->writeln('Not Found: '.$this->matchRoutes(['/foo/bar/baz']));
    }

    private function matchRoutes(array $routes): float
    {
        $startTime = microtime(true);
        foreach ($routes as $route) {
            for ($i = 0; $i < self::NR_OF_MATCHES; ++$i) {
                $this->router->match($route);
            }
        }
        $time = microtime(true) - $startTime;

        return number_format(1000 * $time / count($routes), 2);
    }
{% endhighlight %}

## Results
The table below shows how long it took to match our given routes 50.000 times. 

The __Diff__ column displays how much faster the new Symfony router was, compared to our current one (3.4).  


| Routes(ms)            | 3.4    | 4.1 (7d29a4d) | Diff  |
| ----------------------|--------|---------------|-------|
| First static route    | 448ms  | 382ms         | -17%  |
| Random static route   | 1621ms | 474ms         | -242% | 
| Last static route     | 1826ms | 544ms         | -234% | 
|                       |        |               |       |
| First dynamic route   | 746ms  | 527ms         | -41%  |
| Random dynamic route  | 1454ms | 531ms         | -174% |
| Last dynamic route    | 2039ms | 524ms         | -289% |
|                       |        |               |       |
| Not Found             | 2112ms | 522ms         | -304% | 

As we can see, all of our routes were matched faster than previously. We especially see a significant gain in matching speed for 
the random, last, and when the route was not found.  

### Symfony 3.4
In Symfony 3.4, the matcher has to iterate through all/most of the configured routes and try to find the route for the given url. 
Trying to match a url towards the end of the list will result in an increased matching time, due to the number of comparisons 
being done before the route is found. 

That is why our tests for random route and last route take significantly more time to match than the first route. 
The routes that were not found will take the most amount of time to match, since all the configured routes must be iterated
before drawing the conclusion that our route does not exist. 

### Symfony 4.1
This however is solved nicely in Symfony 4.1. 
Instead of making separate `preg_match()` calls for each route, it combines all the regular expressions into a single regular expression.
This means that we only have to call `preg_match()` once, and that is the biggest factor for faster matching times. 

We are extremely happy with these results, since the only thing we have to do once it is released is to 
run `composer update`.    

Remember that this is our results on our application. 

Your results will probably not be the same since the percentage difference is highly dependant on the amount of routes
that your application has. What results did you get?