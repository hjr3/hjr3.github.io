---
layout: post
title: "Testing Repeated Elements With Behat+Mink"
tags:
- behat
- mink
- symfony 2
- bdd
status: publish
type: post
published: true
---

# Testing Repeated Elements With Behat+Mink

The Mink extension to behat makes it really easy to test the contents of a page. I can use the `assertElementContainsText` feature to assert that some text exists within a certain element:

{% highlight gherkin %}
Then I should see "My Page Title" in the "h1" element.
{% endhighlight %}

If there is more than one `h1` element on the page, I can use a css selector to increase specificity:

{% highlight gherkin %}
Then I should see "My Page Title" in the "h1.page-title" element.
{% endhighlight %}

This is really powerful and css selectors make it pretty easy to identify most elements on a page. However, given the block of html below, how do I test the text in a repeated element?

{% highlight html %}
<div class="items">
    <div class="item-row">
        <p class="item-row-name">Item 1</p>
    </div>
    <div class="item-row">
        <p class="item-row-name">Item 2</p>
    </div>
    <div class="item-row">
        <p class="item-row-name">Item 3</p>
    </div>
</div>
{% endhighlight %}

My initial attempt was to use a more advanced css selector:

{% highlight gherkin %}
Then I should see "Item 1" in the "div.item-row:nth(0)" element.
{% endhighlight %}

Unfortuntely, the Symfony 2 web driver does not support this syntax. After talking with a few colleagues, I decided to create a feature that allowed me to do this. Here is an example of my feature syntax:

{% highlight gherkin %}
Then I should see the following in the repeated "div.item-row-name" element within the context of the "div.items" element:
| text   |
| Item 1 |
| Item 2 |
| Item 3 |
{% endhighlight %}

Here is what the code looks like:

{% highlight php %}
<?php
/**
* @Then /^(?:|I )should see the following in the repeated "(?P<element>[^"]*)" element within the context of the "(?P<parentElement>[^"]*)" element
*/
public function assertRepeatedElementContainsText(TableNode $table, $element, $parentElement)
{
    $parent = $this->getSession()->getPage()->findAll('css', $parentElement);

    foreach ($table->getHash() as $n => $repeatedElement) {
        $child = $parent[$n];

        \PHPUnit_Framework_Assert::assertEquals(
            $child->find('css', $element)->getText(),
            $repeatedElement['text']
        );
    }
}
{% endhighlight %}

I take advantage of the fact that the repeated elements have a common parent `div.items`. I find all children of the `div.items` element using the Mink `find` API. I can loop over the children and take advantage of the fact that the children are of type `NodeElement`. The `NodeElement` class shares the same base class as `DocumentElement` object returned from `$this->getSession()->getPage()` call. When I use the `find` method on the `$child` object, I will only search for elements that are within the context of the current child. Here is what that context looks like on the first iteration:

{% highlight html %}
<div class="item-row">
    <p class="item-row-name">Item 1</p>
</div>
{% endhighlight %}

Now when I search for the element matching `div.item-row-name`, I only get back the _one_ element within this context. I can then assert that this text within this element matches the corresponding item in my table.

Notice that I use a PHPUnit assertion within this feature. I would have preferred to re-use an existing Mink web assertion, but all of the assertions assume a global context. Look at the [elementTextContains]() code to see what I mean.

Happy Hacking!
