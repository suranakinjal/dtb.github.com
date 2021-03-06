---
layout: post
title: Manipulating the caret (text cursor) using the Win32 API
summary: Learn how to manipulate the text caret in an input field using Win32.
styles: "
.spacy {
	border: 1px solid #666;
	padding: 5px 5px;
	background-color: rgba(255, 255, 255, .4);
}
.extra_emph {
	font-style: normal;
	border-bottom: 3px double black;
}
"
---

# Manipulating the caret (text cursor) using the Win32 API

<span class="pubdate">published 19 jun 2011</span>

I recently dealt with a case where I wanted to automatically format user's text as they type it. 
For example, if the user is typing a phone number, they
might type “9595551234”, and as they type, their text would be formatted as such
(system-inserted characters <em class="extra_emph">emphasized</em>):

&#x20;<span class="spacy"><em class="extra_emph">(</em>959<em class="extra_emph">) </em>555<em class="extra_emph">-</em>1234</span>.

This kind of user interface,
when done very carefully [^carefully], can make data entry faster and ensure better-formatted results.

Unfortunately using the `WM_SETTEXT` message to set the text of the control
causes the text caret (aka cursor, aka text insertion point) to revert to the beginning of the text.  We want to 
make sure that the user's cursor ends up where it belongs, so that they can continue to type.

Initial research led me to believe that [`setcursorpos`][scp] and its counterpart [`getcursorpos`][gcp] would do what I wanted,
but after some frustrating trials, I found that while `getcursorpos` would return something vaguely plausible,
`setcursorpos` did nothing at all. I suspect that in fact these functions only work if you are manually managing
cursors you create yourself using [`createcursor`][cc], but I'm not sure.

Fortunately, some [vague whisperings][whisperings]  pointed me to a pair of messages that would do 
the job: [`EM_SETSEL`][setsel] and [`EM_GETSEL`][getsel]. From the `EM_GETSEL` docs:

> If there is no selection, the starting and ending values are both the position of the caret.

I made a trimmed down sample app for this post. Let's say I've got users who love typing “hahahaha” ad infinitum;
By inserting the “a” for them, they could type that string simply by holding down the “h” key, which cuts their keystrokes by 50%! The 
code is as follows:

Inside your handler for the text change notification (see [full solution][fs-context] for more context):

{% highlight c++ %}
// first get the text the user has entered
TCHAR buff[64];
SendMessage(hwndEdit, WM_GETTEXT, 32, (LPARAM)buff); // (32 in case these are wide chars)
size_t len = _tcslen(buff);

// if they've entered text, and the last character is an h
if(len > 0 && buff[len - 1] == 'h')
{
    // also, if we have room in our buffer for the "a"
    if ( len <= 62)
    {
        // add the a
        _tcsncat(buff, L"a", 1);
        
        // before manipulating the text, save their current selection
        DWORD firstChar, lastChar;
        SendMessage(hwndEdit, EM_GETSEL, (WPARAM) &firstChar, (LPARAM) &lastChar);
        
        // now set the text -- guard is a simple mutex to prevent recursing into this 
        // change notification
        gaurd = true;
        SendMessage(hwndEdit, WM_SETTEXT, NULL, (LPARAM) buff);
        gaurd = false;
        
        if(firstChar == lastChar && firstChar == len)
        {
            firstChar++;
            lastChar++;
        }
        
        // restore their cursor position or selection
        SendMessage(hwndEdit, EM_SETSEL, (WPARAM) firstChar, (LPARAM) lastChar);
    }
}
{% endhighlight %}

The key logic here (and this is probably not really sophisticated enough, to be honest—consider a user trying to delete a trailing “a”) 
is that we check if they initially had nothing selected (`firstChar == lastChar`), and if their cursor was at the end of the string 
(`firstChar == len`.) In this case, we want to move the cursor forward by one character to account for the character we just added.

It took me a while to figure this out, and for whatever reason, the documentation is poor, and there's a lot of misleading information 
around the web, so I hope this helps someone!

I've made a simple sample project, which you can find on [Github][sample]. Most of the code there is generated by VS2010's project
template. The real action is in the message handling code in [`MoveCaret2.cpp`][mc2].

[^carefully]: Manipulating the user's text as they enter it is fraught with peril. Be prepared to
do a great deal of testing before you consider this feature ready for prime time. Some questions
to ask:
- What happens if the user enters their own punctuation instead of accepting mine? Are users allowed to override
default formatting rules?
- Am I assuming a certain length of string? What happens if the user enters a different length?
- What happens if the user goes back and edits their text somewhere in the middle?
- What happens if the user deletes their text, or a range of text?

[scp]: http://msdn.microsoft.com/en-us/library/ms648394(v=vs.85).aspx
[gcp]: http://msdn.microsoft.com/en-us/library/ms648390(v=vs.85).aspx
[cc]: http://msdn.microsoft.com/en-us/library/ms648385(v=vs.85).aspx
[whisperings]: http://support.microsoft.com/kb/109550
[setsel]: http://msdn.microsoft.com/en-us/library/bb761661(v=vs.85).aspx
[getsel]: http://msdn.microsoft.com/en-us/library/bb761598(v=vs.85).aspx
[fs-context]: https://github.com/dtb/MoveCaret/blob/master/MoveCaret2/MoveCaret2.cpp#L155
[sample]: https://github.com/dtb/MoveCaret
[mc2]: https://github.com/dtb/MoveCaret/blob/master/MoveCaret2/MoveCaret2.cpp#L134
