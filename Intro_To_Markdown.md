# What is Markdown?

This Markdown document was created by Patrick M. Hooper on 15/01/2024.

# Preamble

Markdown is a lightweight markup language that allows you to add formatting elements to plain text documents. This requires the use of notation to signify formatting. This allows you to include code in a document alongside plain text

To write my markdown files, I use [notepad++](https://notepad-plus-plus.org/) for Windows. This basic text editor contains markdown language formatting which makes writing notes much easier. 

I then upload my markdown files to my GitHub page and track changes through Git in the GitHub Desktop application. This system works for me for text documents as it means I can easily upload my files saved on my PC to GitHub which contains markdown implementation (i.e. formatting) so that I can read my markdown scripts in their formatted form (like you're doing right now)

# Why should we use Markdown?

Markdown allows us to quickly write formatted documents. This is especially useful for formatting our notes when coding or writing script. Markdown files are commonly used for .README files, which should be included in every work project folder. Markdown improve accessibility for other researchers.

Reference: https://www.markdownguide.org/getting-started/ 

1. **Flexibility** - The adaptiveness of Markdown makes it a vital tool for bioinformatics. Markdown files can be used for everything... websites, documents, books, presentations, etc.
2. **Portability** - Markdown can be used on virtually any application. This means you can send Markdown files between software / devices.
3. **Security** - If all else fails, you have your text written in a way that can be read in any editor forever, so even if your software is discontinued it will still be readable.. **and rescuable!**
4. **GitHub** - GitHub uses markdown for its .README files and is common practice for tutorials.
5. **Ease** - It doesn’t take long to learn the Markdown syntax, and once you know how to use it, you can write using Markdown just about everywhere.


# How do we use Markdown?

Using markdown is different from Word editors (e.g. Microsoft Word) as you denote formatting with syntax, e.g. to denote a header you use a # and asterisks to **denote bold**.

## We can **markdown** anywhere, but we need applications to see the formatting!

We save our files in plaintext files with the .md extension. To transfer this to a HTML or print-ready document requires a Markdown application. Markdown processor applications or parser or implementation convert the Markdown-formated text file and output it to HTML format. This output can then be viewed in a web browswer. 

# What do we need to learn in Markdown?
Here's a good place to start: https://www.markdownguide.org/basic-syntax/ 

## 1. Heading

Headers work in a very simple manner, with the # symbol

"# H1"  
"# H2"  
"### H3"  
etc.  

**wait!** it is important to leave a space after your number sign in all headers, this is due to formatting differences between different software. **Always leave a space.**  
**one more thing...**, always leave a space eithe side of a heading and don't use indents on paragraphs, just use a new line to signify a new paragraph.  
***Okay i really thought this was done too***, a double space at the end of a line signifies a line  break.  


## 2. Bold and Italic

Two asterisks either side of your text signifies **bold**  
One asterisk signifies *italics*  
P.s. a strikethrough is specified with two tildes like ~~this~~  
P.p.s highlighting is with the equals sign, ==twice==.  
 

## 3. Blockquote

We use the greater than sign to signify a block quote. ***What's a block quote?***  

A blockquote is a long quotation, set on a new line and indented to create a separate block of text within your text file. (Like this one!)

> Specify a new paragraph in a block quote by adding an empty arrow line
>
> Like this! 
>> We can also nest blockquotes for blockception by adding another greater than sign!


## 4. Listing 

There are two basic types of list you can make in Markdown: Ordered and Unordered. Ordered are numbered lists and unordered are bullet points, signified by the dash symbol  

1. One
2. Two 
3. Three

- Bullet
- Point
- Listing

P.S. if you want to get fancy, we can also write Task Lists
- [x] task one
- [x] task two
- [x] task three

## 5. Code

This is why we're really here, using the code feature of Markdown allows us to specify a chunk of code seperate from our text file. Within this we can specify what language we are coding in, e.g. Python, Bash, R. The benefit of Markdown processor applications is they then format that into boxes of code that are then executable in a command line.  

We need the little used button just above tab and just below esc, called the backtick ('):
 
`2 + 2`

To make a fenced code block we use the slightly more complicated structure:
```
{
Here we insert our code
}
```

## 6. Horizontal Rule

This is a basic formatting tool to seperate a document we specify with three dashes

Next
---
Section

## 7. Links

We can easily insert hyperlinks by typing the title in square brackets and then following it with the link in curved brackets
[Link to my GitHub profile](https://github.com/paddyhooper)

## 8. Images

We can even add images by specifying file pathways
![Tux]{/assets/images/tux.png)

Psst... this also works with URLs
![Find God](https://res.cloudinary.com/practicaldev/image/fetch/s--3OiNiqY7--/c_imagga_scale,f_auto,fl_progressive,h_420,q_auto,w_1000/https://dev-to-uploads.s3.amazonaws.com/i/kc1str971ujjedi3h32i.png)


## 8. More complex syntax

All of this info comes from the [Extended Syntax](https://www.markdownguide.org/extended-syntax/)
Here you can find more like tables, emojis :joy:, subscript and superscript. 

#A great website for previewing your markdown files is dillinger
https://dillinger.io/

I was able to link this to my GitHub account!

- [x]Markdown
- [x]Git
- [x]Slurm (Job Schedulers)
- [x]HPC
- [x]Environments
- [x]Conda