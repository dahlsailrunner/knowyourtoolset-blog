---
title: "Controlling the Width of a Table With Read-the-Docs" # Title of the blog post.
date: 2018-02-18T10:48:49-05:00 # Date of post creation.
description: "Preventing the inclusion of a scroll bar in a ReadTheDocs table - all of the content will show on the page and in the table - just taller." # Description used for search engine.
thumbnail: "/images/better-table.png" # Sets thumbnail image appearing inside card on homepage.
shareImage: "/images/better-table.png" # Designate a separate image for social media sharing.
codeMaxLines: 10 # Override global value for how many lines within a code block before auto-collapsing.
codeLineNumbers: false # Override global value for showing of line numbers within code block.
figurePositionShow: true # Override global value for showing the figure label.
tags:
  - Documentation
  - Read-The-Docs
---

## "But I don’t want a scrollbar!!"
[Read The Docs](https://readthedocs.org/) is a great platform for creating documentation that lets you focus on content rather
than complex markup syntax or making it searchable. One of the elements you might want to put in
your documentation is a table. There are different ways to create a table, and one that I like
is the `list-table` format.

Consider the following markup:

```rst
.. list-table:: Comparison
   :widths: 20 10 10 15 20 
   :header-rows: 1   
 
   * - Platform
     - Self-Contained?
     - Cost
     - Flexibility
     - Description
   * - Raspberry Pi
     - No
     - $30 
     - Limitless
     - Mini computer board with GPIO pins for interfacing and experimentation.
   * - Lego Mindstorms
     - Yes
     - $350
     - Medium
     - Lego robotics sytem with motors and sensors.  Build a robot, then write logic to move it around and do stuff.
  ```

The markup above creates the following table as shown in the following image; ***specifically note the horizontal scrollbar at the bottom***:

![::img-center img-shadow](/images/rtd-table-default.png)

Often, there is good content to the right of what you can initially see, but there is no built-in option to keep the contents of the table directly in the viewing area.

## Custom CSS to the rescue!
A little bit of custom CSS will enable what we’re after, but there are some specific steps to get the custom CSS into a Read The Docs project.

### Create the custom CSS file
If you’ve created a Read The Docs project using `sphinx-quickstart`, you probably have a `_static` directory in your project folder. The image below shows the folder structure and the CSS code that you should add to the file. The class name of `tight-table` can be anything you want, but we will use it again later so keep track of it if you change it.

![::img-center](/images/custom-css-2.png)

### Get the custom CSS into the built output
Just putting the CSS there will not allow you to reference the class yet — it needs to be copied to the HTML built output that gets created when you “make” the project.

To do this, add the following code to you `conf.py` file:
```python
def setup(app):
   app.add_stylesheet('css/custom.css')
```
The path in the above code is relative to the `_static` folder in your doc project. The `setup` function is something that the Read The Docs build process will look for and run automatically if it’s available.

### Reference the class
Back in the original document with your table, you can now simply reference the class as shown in the code snippet below.

```rst
.. list-table:: Comparison
   :widths: 20 10 10 15 20 
   :header-rows: 1
   :class: tight-table   
 
   * - Platform
     . . .
```
## Yay! No more scrollbar!
When you build the project with all of these changes in place, you now get a table that looks as shown below — the content is all visible without scrolling!!!

![::img-center img-shadow](/images/better-table.png)

## Want more??
If you want to know more about Read The Docs, from anything from Sphinx tips, hosting the project on your own or inside [ReadTheDocs.org](https://readthedocs.org/), the Git workflow that goes along with it, versioning, and more, I encourage you to check out my Pluralsight course titled [Build or Contribute to Documentation with a Git-based Workflow](https://app.pluralsight.com/library/courses/build-contribute-documentation-git-based-workflow).