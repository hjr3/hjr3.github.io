+++
title = "Evaluate multidimensional arrays using vim xdebug"
path = "/2012/03/27/evaluate-multidimensional-arrays-using-vim-xdebug.html"

[taxonomies]
tags=["php", "vim", "xdebug"]
+++

I love vim. I love XDebug. I need it to evaluate multidimensional (or nested) arrays though and it does not seem to do that. Chris Hartjes tweeted about his frustration with arrays too and I decided to fix the problem. Turns out there is nothing to fix.

Turns out we need to tweak a default configuration settings to get this all to work. Open up your .vimrc and add the following line:

<code> let g:debuggerMaxDepth = 3
</code>

That depth means you will be able to view the contents of a triply-nested array. That seemed like a sensible default to me. Now you can evaluate or get the property of any variable like normal.

Once the depth is set there is no way to change it during a debug session. You have to close the existing XDebug session, update the value and start a new session. I plan on changing this in a future release of the plugin.
