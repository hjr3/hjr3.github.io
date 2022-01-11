+++
title = "Exploring Facebook's Thrift"
path = "/2009/04/20/exploring-facebooks-thrift.html"
+++

At work we have a number of services written in C.  These services use the same core libraries our legacy C application uses.  Keeping all this business logic in a central place makes life much easier, but the way services are written is horribly inefficient.  I am on a mission to find a better way.

<!-- more -->

All our services communicate using XML with the same basic structure.  Without going into too much detail, there is a single complex tag which contains data specific to that request.  The problem with the way we do services is that each service is responsible for parsing a unique set of XML.  We have over 50,000 lines of service code dedicated to parsing XML.  Maintenance is problematic and making changes to services is often hindered by changes to the client(s), the XML and the service parsing code.  Thankfully many of our existing services are becoming obsolete.  However, we are at a pivotal point where there is a need for a new set of services.  A perfect time to make some changes.

At work there are a number of finance calculations written in C for the legacy application.  As we migrate more and more of the application onto the web, one of the biggest questions is what to do with these calculations.  The current service framework we use has proven too difficult to maintain and re-writing all the calculations in php is an inefficient use of our time and may be too slow.

I had come across Thrift some time ago, but rejected it as it was still pretty new.  Now that it resides in Apache's incubator, I felt it was worth checking out as an alternative to the existing way we do services.  One of the first things I noticed was that Thrift's PHP tutorial seemed to be incomplete.  So I decided to create a small example program to demo Thrift to my co-workers.

Thrift's code generation speaks for itself.  But, how fast is it?  I decided to run a quick comparison between raw php and thrift.  I implemented a basic APR calculation in php 

<pre lang="php" line="1">
<?php

function calc_apr($loanamount, $payment, $term)
{
    $apr = 0;
    $apr_next = .1;
    $tolerance = .00000005;

    //Newton-Raphson method
    while (($apr_next > $apr + $tolerance) or ($apr_next < $apr)) {
        $apr = $apr_next;
        $apr_next = $payment / $loanamount * (pow(1 + $apr, $term) - 1)
            / (pow(1 + $apr, $term));
    }

    return $apr * 1200;
}
</pre>

For a good general case, I used the <a href="http://www.fdic.gov/regulations/laws/rules/6500-1950.html#6500appendixjtopart226">FDIC's</a> example numbers.  The php code performed pretty well.

<pre lang="php" line="19">
echo calc_apr(1000, 33.61, 36), "n";
</pre>
<strong>Result:</strong>
<code>real    0m0.017s
user    0m0.012s
sys     0m0.005s</code>

However, I wanted something that took more processing power to compute.  So I plugged in some numbers to simulate a worst case scenario.  As you can see, the amount of iterations required to calculate this result was costly.
<pre lang="php" line="19">
echo calc_apr(1000, 500.1, 2 ), "n";
</pre>
<strong>Result:</strong>
<code>real    0m0.442s
user    0m0.439s
sys     0m0.003s</code>


Now it is time to build a thrift service.  There is no C support for thrift, so I chose C++ for the server and php for the client.  I created a simple thrift file that defines a simple deal object and a Sale service with a lonely calc_apr method.

<pre lang="cpp" line="1">
namespace cpp Sale
namespace php Sale

struct deal {
  1: i32 dealno
  2: double payment
  3: i32 term
  4: double amtfin
}

service Sale {
   double calc_apr(1:deal d)
}

</pre>

After using the thrift utility to build my C++ and php files, I had to fill in the C++ skeleton with some actual logic.  The function below is pretty much identical to the php one I created above.  The only difference is the use of a deal struct to pass the values in as a single object.

<pre lang="cpp" line="1">
class SaleHandler : virtual public SaleIf {
    public:
        SaleHandler() {
        }

        double calc_apr(const deal& d) {
            double apr = 0;
            double apr_next = .1;
            double range = .00000005;

            while((apr_next > apr + range) || (apr_next < apr)) {
                apr = apr_next;
                apr_next = d.payment / d.amtfin * (pow(1 + apr, d.term) - 1) /
                    (pow(1 + apr, d.term));
            }

            return apr * 1200;
        }
};
</pre>

The php client is so basic I won't even show it.  In the middle of the client skeleton's try block, I created a new Sale_deal() object, assigned the same values used to test the php code and called the calc_apr() function.

I first plugged in the numbers from the general case.  As expected, there is some overhead to all the generated code, but not as much as I thought there would be.

<strong>Result:</strong>
<code>real    0m0.037s
user    0m0.030s
sys     0m0.007s</code>

The big difference is in the worst case.  The performance benefits of a compiled language really start to shine here.

<strong>Result:</strong>
<code>real    0m0.057s
user    0m0.027s
sys     0m0.011s</code>

The worst case scenario may seem unfair, but it represents what is really going on with our finance calculations at work.  The complexity is much higher and there are even iterations within iterations to figure out some of the more obscure numbers.  With this comparison, Thrift appears to be a viable alternative to our current approach to services.
