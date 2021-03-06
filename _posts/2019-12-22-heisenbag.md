---
layout: post
author: alekum
category: Development
title: Heisenbug or the true story of undefined behavior, part 1
olink: <a href="https://alekum.wordpress.com/2016/11/08/heisenbug-the-true-story-of-undefined-behavior-part-1/">Original link</a>
---

One day I was developing a program to work with a python source code. A kind of analyzer for the static code analysis. I found a library <code>Redbaron</code> and wanted conduct some research to understand its capabilities.<code>No so fast cowboy</code> I wrote a lot of python code in <a href="https://www.jetbrains.com/pycharm/"><code>Pycharm</code></a> created by the <a href="https://www.jetbrains.com/"><code>JetBrains</code></a>. After my tests via <a href="http://doc.pytest.org/en/latest/"><code>pytest</code></a> I’ve got the first bug. I’ve got the same problem like in <a href="https://github.com/PyCQA/redbaron/issues/119">#119</a> (Really, Is it a problem of IDE by the JetBrains?).

After debugging, I found some solutions, patches, and hacks.

Next, I’m telling you, dear reader, a story about a difference between hacks, patches, and solutions.

<strong>And Yeah, It hasn’t been a problem of IDE.</strong><!--more-->

Let me start from excerpts of the code.

This part of the code for the version 0.6.1 that’s contained in pip:

<ul>
<li><a href="https://github.com/PyCQA/redbaron/blob/0.6.1/redbaron/base_nodes.py#L1591">https://github.com/PyCQA/redbaron/blob/0.6.1/redbaron/base_nodes.py#L1591</a>**</li>
</ul>

This part of the code for the version of master in git:

<ul>
<li><a href="https://github.com/PyCQA/redbaron/blob/master/redbaron/base_nodes.py#L1642">https://github.com/PyCQA/redbaron/blob/master/redbaron/base_nodes.py#L1642</a>*</li>
</ul>

In master branch, that problem didn’t appear, but the bug was not solved. It wasn’t appearing because last changes have double-negative condition:

<ul>
    <li>If RedBaron was running under debug then in <strong>repr</strong> method of Node class the condition <a href="//github.com/PyCQA/redbaron/blob/master/redbaron/base_nodes.py#L970%2a">if in_a_shell()</a> is True and data flow was passing into <strong>str</strong>().</li>
</ul>

<blockquote>That’s true for the <code>python shell</code> and <code>ipython</code> shell too. And that’s why the bug was not occurring in the shell.</blockquote>

<ul>
    <li>If RedBaron was running without debug mode and not in the shell then the bug didn’t occur because in log function ‘redbaron.DEBUG` parameter is False. And <strong>repr</strong> function was not called. String representation of the object was not caused by the caller.</li>
</ul>

<blockquote>That’s right for the master branch. But for 0.6.1 it isn’t true.</blockquote>

Look carefully at this code in 0.6.1 and in master branch:

<ul>
    <li>the difference was in one comma in call of log function.</li>
</ul>

<strong>The cost of this bug was in one comma.=) But it is actually still deeper, check.</strong>

<blockquote>This problem occurs in 0.6.1 because in <a href="https://github.com/PyCQA/redbaron/blob/0.6.1/redbaron/base_nodes.py#L1591">this code</a> python tries to get a string before it calls a log function. It calls a <strong>repr</strong> function implicitly. In <strong>repr</strong> function <code>in_a_shell()</code> was returned False for Pycharm or something else that’s not a python shell or ipython shell.</blockquote>

Next, it is trying to get a path :

<em><a href="https://github.com/PyCQA/redbaron/blob/0.6.1/redbaron/base_nodes.py#L951">https://github.com/PyCQA/redbaron/blob/0.6.1/redbaron/base_nodes.py#L951</a></em>,

and, after a few intermediate calls, stops here:

<em><a href="https://github.com/PyCQA/redbaron/blob/master/redbaron/base_nodes.py#L121">https://github.com/PyCQA/redbaron/blob/master/redbaron/base_nodes.py#L121</a></em>

, to get an index of the node in the parent node list, but a tree has not

synchronized yet with last changes:

<em><a href="https://github.com/PyCQA/redbaron/blob/0.6.1/redbaron/base_nodes.py#L1333">https://github.com/PyCQA/redbaron/blob/0.6.1/redbaron/base_nodes.py#L1333</a></em>.

Look, here we have already done insertion of the code. <code>Node.data</code> includes insertion, but <code>node.node_list</code> does not. Parent points to <code>node_list</code>but <code>node_list</code> doesn’t have a new insertion. <code>Index</code> method raises the exception - ValueError(<em><a href="https://docs.python.org/2/tutorial/datastructures.html">https://docs.python.org/2/tutorial/datastructures.html</a></em>) because there is no such item.

Look at excerpt from <em><a href="https://github.com/PyCQA/redbaron/blob/master/redbaron/base_nodes.py#L121">https://github.com/PyCQA/redbaron/blob/master/redbaron/base_nodes.py#L121</a></em>:

<strong><code>pos = parent.index(node.node_list if isinstance(node, ProxyList) else node)</code></strong> for <strong><code>parent.index(node)</code></strong> occurred <strong><code>ValueError</code></strong> and message of this error calls <strong>repr</strong> again to get a representation of the object, I mean, that’s call is being called again and would try to get path and

index again. Because the Tree is not being synchronized before log call.

Yeap, <strong>Pycharm(py.test)</strong> tells about it =)

<blockquote>Let’s start the show!</blockquote>

<ul>
    <li>First solution(<em>fast hack</em>)write something like this into <em><a href="https://github.com/PyCQA/redbaron/blob/master/redbaron/utils.py#L33">https://github.com/PyCQA/redbaron/blob/master/redbaron/utils.py#L33</a></em>:</li>
</ul>

<pre class="prettyprint"><code class="language-python hljs "><span class="hljs-keyword">if</span> os.getenv(<span class="hljs-string">'PYCHARM_HOSTED'</span>):
    <span class="hljs-keyword">if</span> os.getenv(<span class="hljs-string">'IPYTHONENABLE'</span>):
        print(<span class="hljs-string">"IPYTHONENABLE %s"</span> % os.getenv(<span class="hljs-string">'IPYTHONENABLE'</span>))
    print(<span class="hljs-string">"PYCHARM_HOSTED : {0}"</span>.format(os.getenv(<span class="hljs-string">'PYCHARM_HOSTED'</span>)))
<span class="hljs-keyword">if</span> <span class="hljs-keyword">not</span> os.getenv(<span class="hljs-string">'PYCHARM_HOSTED'</span>):
    <span class="hljs-keyword">if</span> __IPYTHON__:
        print(<span class="hljs-string">"IPYTHOSHELL %s"</span> % __IPYTHON__)</code></pre>

<blockquote>I’ve tested this code and that’s why it looks how it looks to avoid exceptions and misunderstanding.

But it was a hack. And maybe it was not useful.</blockquote>

<ul>
    <li>Second solution(<em>patch</em>)add <code>try, except</code> block to catch <code>ValueError</code> if an element cannot be found in the list <em><a href="https://github.com/PyCQA/redbaron/blob/master/redbaron/base_nodes.py#L121">https://github.com/PyCQA/redbaron/blob/master/redbaron/base_nodes.py#L121</a></em>:</li>
</ul>

<pre class="prettyprint"><code class="language-python hljs "><span class="hljs-keyword">if</span> isinstance(parent, NodeList):
  <span class="hljs-keyword">try</span>:
     pos = parent.index(node.node_list <span class="hljs-keyword">if</span> isinstance(node, ProxyList) <span class="hljs-keyword">else</span> node)
  <span class="hljs-keyword">except</span> Exception:
     <span class="hljs-keyword">return</span> <span class="hljs-number">0</span></code></pre>

<blockquote>In this part, we catch exception for <code>ValueError</code> in <code>UserList</code> or another unknown exception(but <code>UserList.index</code> raises <code>ValueError</code> like a list). This is the hack too and code is not clean and clear. And this ain’t cool.</blockquote>

<ul>
    <li>Third solution(<em>patch</em>)
Delete <code>log</code> calls in <em><a href="https://github.com/PyCQA/redbaron/blob/master/redbaron/base_nodes.py#L1642">https://github.com/PyCQA/redbaron/blob/master/redbaron/base_nodes.py#L1642</a></em>.

But the error occurred in another place. Or rewrite a logging function… But it has already done because a log function call was changed that’s satisfied declaration of this method. I described it above.</li>
    <li>Fourth solution(<em>solution</em>)</li>
</ul>

<pre class="prettyprint"><code class="language-python hljs "><span class="hljs-comment">#first test, test_insert failed, changes have not synchronized yet</span>
<span class="hljs-function"><span class="hljs-keyword">def</span> <span class="hljs-title">_synchronise</span><span class="hljs-params">(self)</span>:</span> 
<span class="hljs-comment"># TODO(alekum): log method calls __repr__ implicitly to get a self.data representation </span>

    log(<span class="hljs-string">"Before synchronise, self.data = '%s' + '%s'"</span> % (self.first_blank_lines, self.data))
    <span class="hljs-comment">#log("Before synchronise, self.data = '%s' + '%s'", self.first_blank_lines, self.data)</span>

    super(LineProxyList, self)._synchronise()

    <span class="hljs-comment">#log("After synchronise, self.data = '%s' + '%s'" % (self.first_blank_lines, self.data)) </span>
    log(<span class="hljs-string">"After synchronise, self.data = '%s' + '%s'"</span>, self.first_blank_lines, self.data)</code></pre>

<pre class="prettyprint"><code class="language-python hljs "><span class="hljs-comment">#second test, test_insert passed, changes have synchronized already</span>
<span class="hljs-function"><span class="hljs-keyword">def</span> <span class="hljs-title">_synchronise</span><span class="hljs-params">(self)</span>:</span> 
<span class="hljs-comment"># TODO(alekum): log method calls __repr__ implicitly to get a self.data representation </span>

    <span class="hljs-comment">#log("Before synchronise, self.data = '%s' + '%s'" % (self.first_blank_lines,  elf.data)) </span>
    log(<span class="hljs-string">"Before synchronise, self.data = '%s' + '%s'"</span>, self.first_blank_lines, self.data)

    super(LineProxyList, self)._synchronise()

    log(<span class="hljs-string">"After synchronise, self.data = '%s' + '%s'"</span> % (self.first_blank_lines, self.data))
    <span class="hljs-comment">#log("After synchronise, self.data = '%s' + '%s'",self.first_blank_lines, self.data)</span></code></pre>

The solution, that I've described: actual state of the node are in node_list.

<pre class="prettyprint"><code class="language-python hljs "><span class="hljs-function"><span class="hljs-keyword">def</span> <span class="hljs-title">_synchronise</span><span class="hljs-params">(self)</span>:</span> 
<span class="hljs-comment"># TODO(alekum): log method calls __repr__ implicitly to get a self.data representation </span>

    log(<span class="hljs-string">"Before synchronise, self.data = '%s' + '%s'"</span> % (self.first_blank_lines, self.node_list))
    <span class="hljs-comment">#log("Before synchronise, self.data = '%s' + '%s'", self.first_blank_lines, self.data) </span>

    super(LineProxyList, self)._synchronise()

    log(<span class="hljs-string">"After synchronise, self.data = '%s' + '%s'"</span> % (self.first_blank_lines, self.node_list))
    <span class="hljs-comment">#log("After synchronise, self.data = '%s' + '%s'",self.first_blank_lines, self.data)</span></code></pre>

I’ve tested it with master and 0.6.1, all was good.

That was a true solution because <em>it was solving the problem instead of masking them.</em> I think you are a good engineer if you are solving problems and know the difference between hack, patch, and solution.

<blockquote>PS>
Update:
My PR has been applied.
https://github.com/PyCQA/redbaron/pull/122
<blockquote>Thanks for your time, will wait for your comments.</blockquote>
</blockquote>

