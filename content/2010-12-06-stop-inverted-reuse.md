+++
title = "Stop Inverted Reuse"
path = "/2010/12/06/stop-inverted-reuse.html"

[taxonomies]
tags=["design", "inverted reuse", "oo", "php", "reuse"]
+++

Reuse is a term often used amongst developers.  It usually carries with  it a positive connotation and a developer writing reusable code is seen  as a good thing.  I think there are a lot of developers who have a completely different understanding of what code reuse means.  When I talk about code reuse, I am talking about reusing logic within the code.  Based on code reviews, it  seems the most common definition of reuse is: anytime a function or  method is used by two or more callers.  This definition fails to realize  the true meaning of reuse and can lead to problems.  In the name of  "reuse", I have noticed some developers group common code at the application  level by creating functions or methods that solve some pseudo-generic  problem.  I call this type or reuse <em>inverted reuse</em>.<!-- more -->

Inverted  reuse, while initially attractive, often causes a maintenance  nightmare.  A developer notices that they are repeating some common logic over and over (probably through copy and pasting) while writing  some application code and decides to make a common method.  While  preventing copy/paste code is good, it should not be done at the cost of  extensibility and code clarity.  The easiest way to see inverted reuse  is by example.  Consider the following:
<pre lang="php">class Member
{
    public function create(array $params)
    {
        $input = $this->_filter($params, 'create');
        if (!$input) {
            return false;
        }

        // create new member

        return true;
    }

    public function update(array $params)
    {
        $input = $this->_filter($params, 'update');
        if (!$input) {
            return false;
        }

        // update existing member

        return true;
    }

    protected function _filter(array $params, $context)
    {
        $def = array(
            'email' => FILTER_VALIDATE_EMAIL,
            'password' => FILTER_SANITIZE_STRING,
            'name' => FILTER_SANITIZE_STRING,
        );

        if ($context == 'update') {
            $def['id'] = FILTER_VALIDATE_INT;
        }

        $input = filter_var_array($params, $def);

        // name is optional
        if (!$input['email'] || !$input['password']) {
            return false;
        }

        if ($context == 'update') {
            if (!$input['id']) {
                return false;
            }
        }

        return $input;
    }
}
</pre>
The use of the  protected _filter method is a classic example of inverted reuse.  The  email, password and name parameters are common to both the create and update  public methods.  However, the update method needs an additional parameter,  called id, to determine what member to update.  It is quite obvious that the  more parameters the _filter method has to filter, the more complicated  the method becomes.  Some parameters may be only applicable to the create method as well, such as  Google Analytics values.  There are a couple solutions to this particular case of inverted reuse.   One approach would be to only use the _filter method for the common  parameters and to have each public method filter and validate the unique  parameters.  I think this fix still misses the real point of reuse.   The _filter method really does two things: uses filter_input_array and  determines which parameters are required.  This same logic is probably going to be required all over the code.  Consider the following:
<pre lang="php">class Member
{
    public function create(array $params)
    {
        $def = array(
            'email' => FILTER_VALIDATE_EMAIL,
            'password' => FILTER_SANITIZE_STRING,
            'name' => FILTER_SANITIZE_STRING,
        );

        $optional = array('name');

        $f = new Filter;
        $input = $f->filterInput($params, $def, $optional);
        if (!$input) {
            return false;
        }

        // create new member

        return true;
    }

    public function update(array $params)
    {
        $def = array(
            'email' => FILTER_VALIDATE_EMAIL,
            'password' => FILTER_SANITIZE_STRING,
            'name' => FILTER_SANITIZE_STRING,
            'id' => FILTER_VALIDATE_INT,
        );

        $optional = array('name');

        $f = new Filter;
        $input = $f->filterInput($params, $def, $optional);
        if (!$input) {
            return false;
        }

        // update existing member

        return true;
    }
}
</pre>
I am not going to define the Filter class as it really  does not matter how it gets implemented.  There are a number of benefits to moving the logic in the _filter method into a separate class.  Putting this class at the  library level allows the entire application to use it.  It is also much  easier to unit test this class as it does one thing and it does that thing  abstractly.  We can also extend, wrap or compose this class if there is  the need.  This is a truly object oriented approach to solving the  filter problem.

You might notice that both public methods look very similar.  The  $def array is only slightly different in the update method, the  $optional array is exactly the same and the use of the Filter class is  identical.  This is completely fine.   We are reusing the same business  logic inside the Filter class and that is what code reuse is all about.   Each public method's intention is clearly defined this way too.  The  context (create or update) required for filtering is implicit.  Contrast  this with the _filter method which required an explicit context to be  passed around.  I would argue that using an implicit context not only  makes the code easier to understand, it also reduces the chance of  causing bugs.
