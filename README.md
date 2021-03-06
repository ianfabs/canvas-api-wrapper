## Super simple to use
This module wraps the [Canvas Api](https://canvas.instructure.com/doc/api/all_resources.html) handling pagination and throttling. Giving easy access to the main CRUD operations, and making the other more specific calls easier as well.
``` js
// Example: Publish all Modules
const canvas = require('canvas-api-wrapper')
canvas.subdomain = 'example' // default:byui

// Gets the course shell (doesn't send any requests)
const course = canvas.getCourse(19284)

// Gets the data for all modules for this course
await course.modules.get()

// modules inherits from Array, so Array methods work
course.modules.forEach(module => {
	module.published = true
})

// Updates everything in the course that has been changed
await course.update()
```

## Table of contents
- [Get Started](#get-started)
- [Settings](#settings)
- [Debugging](#debugging)
- [Standard Calls](#standard-calls)
- [Wrapped Calls](#wrapped-calls)
	- [Course](#course-extends-item)
	- [Items](#items-extends-array)
	- [Item](#item)
- [Item Gets and Sets](#item-gets-and-sets)

## Get Started
#### Install
```
npm i --save canvas-api-wrapper
```
#### Setup
``` javascript
var canvas = require('canvas-api-wrapper')
//  https://<subdomain>.instructure.com (default: byui)
canvas.subdomain = '<subdomain>' 
```
Authorization:
```js
canvas.apiToken = "<TOKEN>"
```
``` bash
# Powershell
$env:CANVAS_API_TOKEN="<TOKEN>"

# CMD
set CANVAS_API_TOKEN="<TOKEN>"

# Linux & Mac
export CANVAS_API_TOKEN="<TOKEN>"
```

## Settings
This library handles all of the throttling, so that you won't go over your
rate limit. But you may want to tweek the settings to speed it up or slow it 
down
```js
// the subdomain under canvas 
//  https://<subdomain>.instructure.com
canvas.subdomain = 'byui';

// Canvas uses a rate-limit point system to handle spamming. Canvas
// currently fills your account to 700 'points' though subject to change
// then subtracts from your 'points' every time you make a call. If 
// you go below 0 then canvas will start sending you 403 (unauthorized) 
// statuses. So when your account goes under the 'rateLimitBuffer'
// this module will halt the requests until it gets filled back to 
// the 'rateLimitBuffer'. Give it a pretty large buffer, it tends to 
// go quite a ways past the buffer before this module catches it.
canvas.rateLimitBuffer = 300;

// How many to send synchronously at the same time, the higher this
// number, the more it will go over your rateLimitBuffer
canvas.callLimit = 30;

// How much the calls are staggered (milliseconds) especially at the
// beginning so that it doesn't send the callLimit all at the same time
canvas.minSendInterval = 10;

// After it goes under the rateLimitBuffer, how often (in milliseconds) 
// to check what the buffer is at now, this should be pretty high because
// there will be a lot of threads checking at the same time.
canvas.checkStatusInterval = 2000;
```

## Debugging

This library ambiguates which calls it's making quite a bit, especially with the [wrapped calls](#wrapped-calls). So to make sure it's not making excess calls. Or to just making sure it's doing what it should be you can assign `canvas.oncall` a function as an event handler.
``` js
canvas.oncall = function(e){
	console.log(e)
}
await canvas.getCourse(11310).pages.create({
	title:'Hello World',
	body:'<h1>Hello World</h1>',
	published: true
})
/*
{ method: 'POST',
  url: 'https://byui.instructure.com/api/v1/courses/11310/pages/',
  body:
   { wiki_page:
      { title: 'Hello World',
        body: '<h1>Hello World</h1>',
	published: true } } }
*/
```

## Standard Calls
Use awaits or the optional callback
```js
var modules = await canvas('/api/v1/courses/10698/modules')

/* ----------- OR -------------- */

canvas('/api/v1/courses/10698/modules', function(err,modules){
	if(err) {
		console.error(err);
		return 
	}
	console.log(modules)
})
```
If a post, put, or delete request the second argument is the request body
```js
await canvas.post('/api/v1/courses/10698/modules',{
	module:{
		name:"New Module"
	}
})
```
If a get request, the second argument is the query object
``` js
var queriedModules = await canvas('/api/v1/courses/10698/modules',{
		search_term:'New Module'
})
```

### Signatures
``` js
canvas(url[,query object][,callback]) // uses a GET request

canvas.get(url[,query object][,callback])

canvas.post(url[,body][,callback])

canvas.put(url[,body][,callback])

canvas.delete(url[,body][,callback])

canvas.getCourse(id)
```

## Wrapped Calls
The CRUD operations for files, folders, assignments, discussions, modules, pages, and quizzes are wrapped for convenience. The can be accessed through the [Course Class](#course-extends-item) which is created through `canvas.getCourse(id)`

### Course _extends_ [Item](#item)
 - `files` <[Files]>
 - `folders` <[Folders]>
 - `assignments` <[Assignments]>
 - `discussions` <[Discussions]>
 - `modules` <[Modules]>
 - `pages` <[Pages]>
 - `quizzes` <[Quizzes]>
 
``` js
var course = canvas.getCourse(17829)

// this will get the course object and attach the properties to the course
await course.get()
// allowing you to do 
course.getTitle()
course.is_public = true
await course.update()

// using getComplete, will retrieve every single item and 
// their sub items in the course. This is not recommended (of course) but 
// can be helpful in certain situations where you need every item
await course.getComplete()
course.quizzes[0].questions[0].getTitle()
course.modules[0].moduleItems[0].getTitle()
course.assignments[0].getTitle()

// Update searches through all of the children for changes, and updates
// only those with changes. So just doing course.update() will push all
// changes made anywhere in the course.
await course.update()
```

### Items _extends_ **Array**

The abstract class which all of the lists of items inherit from

- _async_ `update( [callback] )`
	- Updates all of the items that have had changes made to them or their children

``` js
const course = canvas.getCourse(19823)
await course.assignments.get()
course.assignments.forEach(assignment => {
	if(assignment.getTitle() == 'potato'){
		assignment.setTitle('Baked Potato')
		assignment.published = true
	}
})
// Only updates items named potato, because those were the only ones changed
await course.assignments.update()
```

- _async_ `create( data, [callback] )` <[Item](#item)>
	- Creates the item in canvas, with the given data. And adds it to the items property
	- `data` <**Object**> the properties to add to the created item

``` js
const course = canvas.getCourse(19823)
const page = await course.pages.create({
	title:'Hello World',
	body:'<h1>Hello World</h1>',
	published: true
})
console.log(page.getId())
```

- _async_ `get( [callback]  )` <**[[Item](#item)]**>
	- Retrieves all of the children items from canvas
``` js
const course = canvas.getCourse(19823)
// you don't have to save it to a variable, 
// you can also just access the array through `course.modules`
// this just saves you from writing `course` everywhere
const modules = await course.modules.get()
console.log(modules)
```

- _async_ `getComplete( [callback]  )` <**[[Item](#item)]**>
	- Retrieves all of the children items from canvas and all of their sub items such as the `questions` in a `quiz` or the `body` in a `page`
``` js
const course = canvas.getCourse(19823)
const modules = await course.modules.getComplete()
// normally you would have to call `get()` on each of the
// modules ( modules[0].get() ) in order to get the moduleItems
console.log(modules[0].moduleItems)
```

- _async_ `getOne( id, [callback] )` <[Item](#item)>
	- Retrieves a single item from canvas by id
	- `id` <**number**> the id of the item to grab
``` js
const course = canvas.getCourse(19823)
const folder = course.folders.getOne(114166)
console.log(folder)
```

- _async_ `getOneComplete( id, [callback] )` <[Item](#item)>
	- Retrieves a single item from canvas by id including all of it's sub items such as the `questions` in a `quiz` ( the `body` in `page` is already included because your only getting a single item )
	- `id` <**number**> the id of the item to grab
``` js
const course = canvas.getCourse(19823)
const quiz = course.quizzes.getOneComplete(114166)
console.log(quiz.questions[0])
```

- _async_ `delete( id , [callback] )`
	- Removes an item from canvas, and from the local list
	- `id` <**number**> the id of the item to delete
``` js
const course = canvas.getCourse(19823)
await course.quizzes.get()
const questions = await course.quizzes[0].questions.get()
await questions.delete(questions[0].getId())
```
### Item

The abstract class for the items to inherit from

- `getId()` <**number**>
- `getTitle()` <**string**>
- `setTitle( title )`
- `getHtml()` <**string**>
- `setHtml( html )`
- `getUrl()` <**string**>
- `getSubs()` <**[Items]**> - Array of children [Items](#items-extends-array)
- _async_ `get( [callback] )` <[Item](#item)>
	- Retrieves the item from canvas
- _async_ `getComplete( [callback] )` <[Item](#item)>
	- Retrieves the item from canvas including the sub items such as the `questions` in a `quiz` ( the `body` in `page` is already included because your only getting a single item )
- _async_ `update( [callback] )`
	- Only updates if properties have been changed on the Item since it was last gotten, also updates all of it's sub children who have been changed
- _async_ `delete( [callback] )`
	- Will delete item, and remove it from it's parent list

### Assignments _extends_ [Items](#items-extends-array)
### Assignment _extends_ [Item](#item)

### Discussions _extends_ [Items](#items-extends-array)
### Discussion _extends_ [Item](#item)

### Files _extends_ [Items](#items-extends-array)
- Doesn't have a create method
### File _extends_ [Item](#item)

### Folders _extends_ [Items](#items-extends-array)
### Folders _extends_ [Item](#item)

### Modules _extends_ [Items](#items-extends-array)
### Module _extends_ [Item](#item)
- `moduleItems` <**ModuleItems**>
### ModuleItems _extends_ [Items](#items-extends-array)
### ModuleItem _extends_ [Item](#item)

### Pages _extends_ [Items](#items-extends-array)
### Page _extends_ [Item](#item)

### Quizzes _extends_ [Items](#items-extends-array)
### Quiz _extends_ [Item](#item)
- `questions` <**QuizQuestions**>
### QuizQuestions _extends_ [Items](#items-extends-array)
### QuizQuestion _extends_ [Item](#item)


## Item Gets and Sets
| Type | getId | getTitle/setTitle | getHtml/setHtml | getUrl | getSubs |
|------------|-------|------|-----|------|---|
| Course | id | name | | /courses/<_course_> | [ files<[Files]>, folders<[Folders]>, assignments<[Assignments]>, discussions<[Discussions]>, modules<[Modules]>, pages<[Pages]>, quizzes<[Quizzes]> ] |
| Assignment | id | name | description | html_url | |
| Discussion | id | title | message | html_url | |
| File | id | display_name |  | /files/?preview=<_id_> | |
| Folder | id | name | | folders_url | |
| Module | id | name | | /modules#context_module_<_id_> | [ moduleItems<[ModuleItems]> ] |
| ModuleItem | id | title |  | html_url | |
| Page | page_id | title | body | html_url | |
| Quizzes | id | title | description | html_url | [ questions<[QuizQuestions]> ] |
| QuizQuestion | id | question_name | question_text | /quizzes/<_quiz_>/edit#question_<_id_> | |

[Files]: #files-extends-items "Files"
[Folders]: #folders-extends-items "Folders"
[Assignments]: #assignments-extends-items "Assignments"
[Discussions]: #discussions-extends-items "Discussions"
[Modules]: #modules-extends-items "Modules"
[ModuleItems]: #moduleitems-extends-items "ModuleItems"
[Pages]: #pages-extends-items "Pages"
[Quizzes]: #quizzes-extends-items "Quizzes"
[QuizQuestions]: #quizquestions-extends-items "QuizQuestions"