# My Encounter, Analysis, and Solution to The TokenMismatchException Problem in Laravel 5.*#


## *1. My Code* ##
Here is the laravel login process code:

     <form class="form-horizontal" role="form" method="POST" action="/auth/login">
						<input type="hidden" name="_token" value="{{ csrf_token() }}">

And another ajax code of transfering data:

     $(document).ready(function(){
	    $('.send-form-ajax').submit(function(event){
	      $('.send-btn').fadeOut(300);
	      event.preventDefault();
	      $.ajax({
	        url: 'home/editUser',
	        type: "post",
	        data: { 'name':$('input[name="name"]').val(),
	                'company':$('input[name="company"]').val(),
	                'department':$('input[name="department"]').val(),
	                'isLocked':$('input[name="isLocked"]').is(':checked'),
	                '_token':$('input[name="_token"]').val()
	               },
	        success: function(data){
	            alert('ajax 请求成功');
	        },
	        error: function(e) {
	        	//called when there is an error
	        	//console.log(e.message);
	        	alert('ajax 请求失败');
	        }
	      });
	    });
    });

## *2. The Error* ##
  I use the Laravel 5.1 framework to build an application to manage my IP resources. While I get to authentication process, I encount the annoying TokenMismatch Exception. Below is the Exception:
![](https://github.com/wyl206/Web/blob/master/laravel/pic/loginwindow.JPG?raw=true)
  While clicking  the login button, the exception appears.
![](https://github.com/wyl206/Web/blob/master/laravel/pic/tokenMismatch.JPG?raw=true)


## *3. The Environment* ##
  To help others locate their problem, following is my Server configuration.
![](https://github.com/wyl206/Web/blob/master/laravel/pic/apacheConfig1.JPG?raw=true)
![](https://github.com/wyl206/Web/blob/master/laravel/pic/apacheConfig2.JPG?raw=true)
![](https://github.com/wyl206/Web/blob/master/laravel/pic/apacheConfig3.JPG?raw=true)


## *4. The Analysis* ##
  I directly go the VerifyCsrfToken.php to check what is happening there. I set a breakpoint and send a request from my browser.

![](https://github.com/wyl206/Web/blob/master/laravel/pic/tokenCatch.JPG?raw=true)

  I watch the token transfered from request and session, and their are really not equated.

![](https://github.com/wyl206/Web/blob/master/laravel/pic/tokenCompare.JPG?raw=true)

  The token of html is correctedly sent to the request.

![](https://github.com/wyl206/Web/blob/master/laravel/pic/tokenFromHtml.JPG?raw=true)

**So the problem is actually caused by the session! We know that Laravel will start a new session if there is no cookie received. Here is how it's done.**
![](https://github.com/wyl206/Web/blob/master/laravel/pic/startSession.JPG?raw=true)

![](https://github.com/wyl206/Web/blob/master/laravel/pic/getSession.JPG?raw=true)

We can know from that the session tries to get cookies from request, while the cookie is sent from browser. The Server will jot down session information including CSRF token into files or database based on you drivers.

![](https://github.com/wyl206/Web/blob/master/laravel/pic/sessionLog.JPG?raw=true)

But my browser doesn't have any cookies to send:
![](https://github.com/wyl206/Web/blob/master/laravel/pic/siteCookie.JPG?raw=true)

If Laravel doesn't get cookie from request, it will generate a new session ID  and new CSRF token within next steps. Laravel addes these new cookies and headers to the response of the request, then send them to browser. But now the value of ($request->session()->token()) is false due to there is no value and new session has not been started yet.
![](https://github.com/wyl206/Web/blob/master/laravel/pic/tokenCompare.JPG?raw=true)

**Why doesn't the Server generate cookie and send to browser within precedent requests?**

So we have to trace how the Laravel generates and sends cookie. The laravel generates response class within following processing request and then sends the response. We can know that from index.php.
![](https://github.com/wyl206/Web/blob/master/laravel/pic/sendResponse.JPG?raw=true)

![](https://github.com/wyl206/Web/blob/master/laravel/pic/send.JPG?raw=true)

![](https://github.com/wyl206/Web/blob/master/laravel/pic/sendHeader.JPG?raw=true)

**The result is that the Headers have already been sent, but I don't send any headers manually.**
![](https://github.com/wyl206/Web/blob/master/laravel/pic/HeadersSend.JPG?raw=true)

Since the information shows the file  /var/www/html/laravel/storage/framework/views/897f5b2e7a86661b6f3be95ce19b662e has already done those creepy things, I checked the file. It is a compiled view file of outputing my login HTML code. It donesn't do anything like those. 
![](https://github.com/wyl206/Web/blob/master/laravel/pic/compiledView.JPG?raw=true)

I really spend much times digging why the view file will send headers without any code like  sending headers?
I google the reason online and the two sites enlights me up.
[https://docs.joomla.org/Cannot_modify_header_information_-_headers_already_sent](https://docs.joomla.org/Cannot_modify_header_information_-_headers_already_sent) and 
[http://digitalpbk.com/php/warning-cannot-modify-header-information-headers-already-sent](http://digitalpbk.com/php/warning-cannot-modify-header-information-headers-already-sent)

In the HTTP protocol a server response consists of a group of headers followed by a body, separated by a single blank line (i.e. a line containing only a carriage-return). This warning message is produced by PHP if a program attempts to send an additional HTTP header after the separator (and all the headers) has already been sent.

**So I have to dig How the view was processed?  Another interesting thing is that the content of the response is empty. If normal, the content will be HTML code string of view. If it is empty, why the site showes normally?**

![](https://github.com/wyl206/Web/blob/master/laravel/pic/sentContent.JPG?raw=true)

After a long track of the process of response and view, we finally get to the core code of dealing view.

![](https://github.com/wyl206/Web/blob/master/laravel/pic/setContent.JPG?raw=true)

![](https://github.com/wyl206/Web/blob/master/laravel/pic/renderContent.JPG?raw=true)

![](https://github.com/wyl206/Web/blob/master/laravel/pic/evaluatePath.JPG?raw=true)

We can find "include $__path" in the view rendering. while the variable $__path is compiled view of current site.
![](https://github.com/wyl206/Web/blob/master/laravel/pic/compiledView.JPG?raw=true)
It contains another view rendering and this process will go agian. It is a iterated process. In Laravel we can extend parent view from child view. So the process is that we render child view first, then the parent view, and return HTML code in last step of render. 

**How can we return HTML code string in the iterated process?**

From the function evaluatePath the answer is output_buffering, so the ob_start, ob_get_clean, ob_get_level functions.  And these functions are evil we know that. So I check if these functions have been correctedly executed. So I add two lines to check.

![](https://github.com/wyl206/Web/blob/master/laravel/pic/checkOb.JPG?raw=true)

The first level of iteration:

![](https://github.com/wyl206/Web/blob/master/laravel/pic/firstIteration.JPG?raw=true)

The second level of iteration:

![](https://github.com/wyl206/Web/blob/master/laravel/pic/secondIteration.JPG?raw=true)

The result of iteration:

![](https://github.com/wyl206/Web/blob/master/laravel/pic/obResult.JPG?raw=true)

**We can see the $contents of result returned is empty. And the output_buffering level is different before "include $__path" and after "include $__path". In the first level of iteration, the $obOutLevel is 0 before the code  "return ltrim(ob_get_clean())" is executed.**
I remember there is a <?php echo ... ?> in the child compiled view file.
![](https://github.com/wyl206/Web/blob/master/laravel/pic/compiledView.JPG?raw=true)

**So it occurs to me this "echo" code  in the child compiled view file is executed on the Output Buffering Level 0. The answer is clearly that content of echo in output buffering 0 is directly sent to browser. So There is nothing in the output buffering. If the contents of echo-<html> ... </html>-is sent to browser precedingly,  How can the headers be sent back to broswer in the following steps of sending response ?  That is why "the Headers have already been sent" error has gotten happened and the site shows fine. Then it triggers a series of things, cookie not sent, token mismatched.**

**And God knows why it jumps out OB level after executed include $__path, i.e. the compiled view file. I have observed this phenomenon for some times. This thing doesn't always happen related to different views. Like others this thing is random some times, So is the Token Mismatch Exception. I will find that later. **


## *5. My Solution* ##
The solution is simple. That is to maintain it is in the same output buffering level after excuted "include $__path". So I change the evaluatePath function in file "Illuminate\View\Engines\PhpEngine.php".
Here is my solution Code:

    protected function evaluatePath($__path, $__data)
    {

        $obLevel = ob_get_level();
        extract($__data);
        ob_start();
        $obStartLevel = ob_get_level();

        // We'll evaluate the contents of the view inside a try/catch block so we can
        // flush out any stray output that might get out before an error occurs or
        // an exception is thrown. This prevents any partial views from leaking.
        try {

            include $__path;

        } catch (Exception $e) {
            $this->handleViewException($e, $obLevel);
        } catch (Throwable $e) {
            $this->handleViewException(new FatalThrowableError($e), $obLevel);
        }

        //最终是要保证obEndLevel和obStartLevel在同一层
        $obEndLevel = ob_get_level();
        while($obEndLevel > $obStartLevel){
            ob_end_flush(); 
            $obEndLevel = ob_get_level();
        }
        $myContent = ltrim(ob_get_contents());
        while($obEndLevel < $obStartLevel){
            ob_clean();  
            if(!ob_start())  break; 
            $obEndLevel = ob_get_level();
        }
        if($obEndLevel === $obStartLevel) ob_end_clean();
        return $myContent;
    }

Then It works well. The token mismatch exception doesn't appear anymore.
![](https://github.com/wyl206/Web/blob/master/laravel/pic/success1.JPG?raw=true)
![](https://github.com/wyl206/Web/blob/master/laravel/pic/success2.JPG?raw=true)
![](https://github.com/wyl206/Web/blob/master/laravel/pic/success3.JPG?raw=true)

## *6. The Author* ##

	Yiliang Wang, China. Email:wangyiliang206@163.com  








































