
## case 1:

The parameter based to preview digest @ app/controllers/admin/email_controller.rb#preview_digest

Of which it calles 

>     `renderer = Email::Renderer.new(UserNotifications.digest(user, since: params[:last_seen_at]))`

Now Usernotification return an error from 
app/mailers/user_notifications.rb#L246

When calling: `topics_for_digest = Topic.for_digest(user, @since, digest_opts)`

Back to rendere which is creating create an isntance of error, it would through once the controller tries ot access its `html` or `text` in line 

> `render json: MultiJson.dump(html_content: renderer.html, text_content: renderer.text)`


In the of the stack the code that is probablmitic is this snippest 

```
  def self.best_periods_for(date, default_period = :all)
    return [default_period, :all].uniq unless date

    periods = []
    periods << :daily if date > (1.week + 1.day).ago
    periods << :weekly if date > (1.month + 1.week).ago
    periods << :monthly if date > (3.months + 3.weeks).ago
    periods << :quarterly if date > (1.year + 1.month).ago
    periods << :yearly if date > 3.years.ago
    periods << :all
    periods
  end
end
```
From app/controllers/list_controller.rb#L520:#L530

The error is 

```
ArgumentError: comparison of String with ActiveSupport::TimeWithZone failed (ArgumentError)

    periods << :daily if date > (1.week + 1.day).ago
                                ^^^^^^^^^^^^^^^^^^^^
from /workspace/discourse/app/controllers/list_controller.rb:525:in `>'
```

 In summary in this case the error, is when using comparsion opreator to compare date, as rails failed to recongnize parse date in Arabic numerls


 ## case 2 
  
   When embedding a post with arabic date, it's probaly both frontend and backend issue,
   
   Frontend aspect: as also javascript cannnot  parse in Arbaic literlars e.g. `new Date(٢٠٢٥)` would fail with invalid date but `new Date(2025)` is okay.

   backend: the `Post.cook` fails to render which probably as well use/rely on JS as above. 

  ###  stack 
 - Relate line code: app/models/post.rb#L322:#L371. (the defincaiton of hte cook function).
  - Which above relies on `PostAnalyzer`,to repliacte a post with arabic literlas `PostAnalyzer.new(raw8,8).cook(raw8)`
  - Which in return calls `PrettyText(c8)` (last cascase if cooking method not specifed) At this case **Invalid Date** would be rendere
  - PrettyText.cook, relies on it's `self.markdown` function, which replicate the same result.
  - In the end it probably calls v8 engine so that some JS is running. 
  - Which explain how it's a problamitc form backend, while the is JS related.
  - Ref: lib/pretty_text.rb#L168:L241 note `context.eval is v8.eval()
   - Well it truns out, v8.eval(), is using a `      buffer << "__pt = __DiscourseMarkdownIt.withCustomFeatures(__pluginFeatures).withOptions(__optInput);"` hence last line `__pt.cook`.
   - Thus our target become now `__DiscourseMarkdownIt`
 - src of `DiscourseMarkdownIt` is app/assets/javascripts/discourse-markdown-it/src/engine.js 
   - There are sanitze and render, of which sanize relies of the `xss` npm pkg 
   - render relies on DiscousrseMarkdownIt impelemeanetion of markdownIt
   - Still unknown at which steps [date=] is transfered to an HTML object. 
   - probably need to look at someplugin (dataAttribtes) that transfer date using 
    - new Date().


(Hard to redproduce)
- if the cooking method is email it would **thorugh an error** instead of invlaid date `EmailCook.new(c8).cook()`



