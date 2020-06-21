---
layout:     post                    # 使用的布局（不需要改）
title:      java并发编程之CyclicBarrier工具及论文爬取介绍   # 标题 
subtitle:   论文爬取 					#副标题
date:       2020-06-21              # 时间
author:     BY tang                     # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - java
    - 多线程
---

# java并发编程之CyclicBarrier工具及论文爬取介绍 #
> 进行并发编程，CountDownLatch，CyclicBarrier和Semaphore这三个辅助类能够帮助我们完成一些复杂的业务需求。

本篇文章介绍的是CyclicBarrier用法，使用的场景是论文爬取的处理。因为最近需要爬取wos网的引频次数，每次都是爬到中途就报错403 forbidden，经过一番排查发现是**wos的sid参数使用的次数或者是过一段时间就失效**，因此现在需求就是每经过一段时间或者每爬取一定次数的论文后就需要去请求sid，这时候使用CyclicBarrier就符合需求。

## 简介 ##
CyclicBarrier可以使一定数量的线程反复地在栅栏位置处汇集。当线程到达栅栏位置时将调用await方法，这个方法将阻塞直到所有线程都到达栅栏位置。如果所有线程都到达栅栏位置，那么栅栏将打开，此时所有的线程都将被释放，而栅栏将被重置以便下次使用。

CyclicBarrier的类图如下：

![](https://i.niupic.com/images/2020/06/21/8hSe.png)

通过类图我们可以看到，CyclicBarrier内部使用了ReentrantLock和Condition两个类。

它有两个构造函数：

    public CyclicBarrier(int parties) {
        this(parties, null);
    }
     
    public CyclicBarrier(int parties, Runnable barrierAction) {
        if (parties <= 0) throw new IllegalArgumentException();
        this.parties = parties;
        this.count = parties;
        this.barrierCommand = barrierAction;
    }

CyclicBarrier默认的构造方法是CyclicBarrier(int parties)，其参数表示屏障拦截的线程数量，每个线程使用await()方法告诉CyclicBarrier我已经到达了屏障，然后当前线程被阻塞。

CyclicBarrier的另一个构造函数CyclicBarrier(int parties, Runnable barrierAction)，用于线程到达屏障时，优先执行barrierAction，方便处理更复杂的业务场景。从wos请求sid使用的就是这个方法。

## await方法 ##
调用await方法的线程告诉CyclicBarrier自己已经到达同步点，然后当前线程被阻塞。直到parties个参与线程调用了await方法，CyclicBarrier同样提供带超时时间的await和不带超时时间的await方法：

	public int await() throws InterruptedException, BrokenBarrierException {
		try {
		// 不超时等待
			return dowait(false, 0L);
			} catch (TimeoutException toe) {
			throw new Error(toe); // cannot happen
		}
	}

    public int await(long timeout, TimeUnit unit)
        throws InterruptedException,
                BrokenBarrierException,
                TimeoutException {
        return dowait(true, unit.toNanos(timeout));
    }

这两个方法最终都会调用dowait(boolean, long)方法，它也是CyclicBarrier的核心方法，该方法定义如下：

	private int dowait(boolean timed, long nanos)
	    throws InterruptedException, BrokenBarrierException,
	            TimeoutException {
	    // 获取独占锁
	    final ReentrantLock lock = this.lock;
	    lock.lock();
	    try {
	        // 当前代
	        final Generation g = generation;
	        // 如果这代损坏了，抛出异常
	        if (g.broken)
	            throw new BrokenBarrierException();
	 
	        // 如果线程中断了，抛出异常
	        if (Thread.interrupted()) {
	            // 将损坏状态设置为true
	            // 并通知其他阻塞在此栅栏上的线程
	            breakBarrier();
	            throw new InterruptedException();
	        }
	 
	        // 获取下标
	        int index = --count;
	        // 如果是 0，说明最后一个线程调用了该方法
	        if (index == 0) {  // tripped
	            boolean ranAction = false;
	            try {
	                final Runnable command = barrierCommand;
	                // 执行栅栏任务
	                if (command != null)
	                    command.run();
	                ranAction = true;
	                // 更新一代，将count重置，将generation重置
	                // 唤醒之前等待的线程
	                nextGeneration();
	                return 0;
	            } finally {
	                // 如果执行栅栏任务的时候失败了，就将损坏状态设置为true
	                if (!ranAction)
	                    breakBarrier();
	            }
	        }
	 
	        // loop until tripped, broken, interrupted, or timed out
	        for (;;) {
	            try {
	                 // 如果没有时间限制，则直接等待，直到被唤醒
	                if (!timed)
	                    trip.await();
	                // 如果有时间限制，则等待指定时间
	                else if (nanos > 0L)
	                    nanos = trip.awaitNanos(nanos);
	            } catch (InterruptedException ie) {
	                // 当前代没有损坏
	                if (g == generation && ! g.broken) {
	                    // 让栅栏失效
	                    breakBarrier();
	                    throw ie;
	                } else {
	                    // 上面条件不满足，说明这个线程不是这代的
	                    // 就不会影响当前这代栅栏的执行，所以，就打个中断标记
	                    Thread.currentThread().interrupt();
	                }
	            }
	 
	            // 当有任何一个线程中断了，就会调用breakBarrier方法
	            // 就会唤醒其他的线程，其他线程醒来后，也要抛出异常
	            if (g.broken)
	                throw new BrokenBarrierException();
	 
	            // g != generation表示正常换代了，返回当前线程所在栅栏的下标
	            // 如果 g == generation，说明还没有换代，那为什么会醒了？
	            // 因为一个线程可以使用多个栅栏，当别的栅栏唤醒了这个线程，就会走到这里，所以需要判断是否是当前代。
	            // 正是因为这个原因，才需要generation来保证正确。
	            if (g != generation)
	                return index;
	            
	            // 如果有时间限制，且时间小于等于0，销毁栅栏并抛出异常
	            if (timed && nanos <= 0L) {
	                breakBarrier();
	                throw new TimeoutException();
	            }
	        }
	    } finally {
	        // 释放独占锁
	        lock.unlock();
	    }
	}

dowait(boolean, long)方法的主要逻辑处理比较简单，如果该线程不是最后一个调用await方法的线程，则它会一直处于等待状态，除非发生以下情况：

-     最后一个线程到达，即index == 0
-     某个参与线程等待超时
-     某个参与线程被中断
-     调用了CyclicBarrier的reset()方法。该方法会将屏障重置为初始状态

注意Generation 对象，在上述代码中总是可以看到抛出BrokenBarrierException异常，那么什么时候抛出异常呢？如果一个线程处于等待状态时，如果其他线程调用reset()，或者调用的barrier原本就是被损坏的，则抛出BrokenBarrierException异常。同时，任何线程在等待时被中断了，则其他所有线程都将抛出BrokenBarrierException异常，并将barrier置于损坏状态。

同时，Generation描述着CyclicBarrier的更新换代。在CyclicBarrier中，同一批线程属于同一代。当有parties个线程到达barrier之后，generation就会被更新换代。其中broken标识该当前CyclicBarrier是否已经处于中断状态。

    private static class Generation {
        boolean broken = false;
    }

默认barrier是没有损坏的。当barrier损坏了或者有一个线程中断了，则通过breakBarrier()来终止所有的线程：

    private void breakBarrier() {
        generation.broken = true;
        count = parties;
        trip.signalAll();
    }

在breakBarrier()中除了将broken设置为true，还会调用signalAll将在CyclicBarrier处于等待状态的线程全部唤醒。

当所有线程都已经到达barrier处（index == 0），则会通过nextGeneration()进行更新换地操作，在这个步骤中，做了三件事：唤醒所有线程，重置count，generation：

    private void nextGeneration() {
        // signal completion of last generation
        trip.signalAll();
        // set up next generation
        count = parties;
        generation = new Generation();
    }

在使用CyclicBarrier时还要要注意以下几点：

- CyclicBarrier使用独占锁来执行await方法，并发性可能不是很高
- 如果在等待过程中，线程被中断了，就抛出异常。但如果中断的线程所对应的CyclicBarrier不是这代的，比如，在最后一次线程执行signalAll后，并且更新了这个“代”对象。在这个区间，这个线程被中断了，那么，JDK认为任务已经完成了，就不必在乎中断了，只需要打个标记。该部分源码已在dowait(boolean, long)方法中进行了注释。
- 如果线程被其他的CyclicBarrier唤醒了，那么g肯定等于generation，这个事件就不能return了，而是继续循环阻塞。反之，如果是当前CyclicBarrier唤醒的，就返回线程在CyclicBarrier的下标。完成了一次冲过栅栏的过程。该部分源码已在dowait(boolean, long)方法中进行了注释。

## 应用程序示例 ##
paperList为要爬取的论文列表，iSearch.getSID()为获取wos的sid方法。为了保证sid的可见性，sid定义为对象。例子中是每过60秒就会去请求一次sid，输出这个sid会发现每次请求的都是一样的值，因此猜测这个sid请求是为了维持其存活。论文爬取根据的是每篇论文的poi来检索。

	//Package A
	public class JcrTest {
		AtomicInteger integer =  new AtomicInteger(0);
		AtomicReference<String> sid = new AtomicReference<String>();
		private long start;
		ISearch iSearch = SpringContextUtil.getBean(WebOfScienceSearch.class);
	
		public void test(List<Paper> paperList) throws NullPointerException, IOException, URISyntaxException{
			sid.set(iSearch.getSID());
			start= System.currentTimeMillis();
			String url="http://apps.webofknowledge.com/WOS_GeneralSearch.do?fieldCount=1&action=search&product=WOS&search_mode=GeneralSearch&max_field_count=25&max_field_notice=%E6%B3%A8%E6%84%8F%3A+%E6%97%A0%E6%B3%95%E6%B7%BB%E5%8A%A0%E5%8F%A6%E4%B8%80%E5%AD%97%E6%AE%B5%E3%80%82&input_invalid_notice=%E6%A3%80%E7%B4%A2%E9%94%99%E8%AF%AF%3A+%E8%AF%B7%E8%BE%93%E5%85%A5%E6%A3%80%E7%B4%A2%E8%AF%8D%E3%80%82&exp_notice=%E6%A3%80%E7%B4%A2%E9%94%99%E8%AF%AF%3A+%E4%B8%93%E5%88%A9%E6%A3%80%E7%B4%A2%E8%AF%8D%E5%8F%AF%E4%BB%A5%E5%9C%A8%E5%A4%9A%E4%B8%AA%E5%AE%B6%E6%97%8F%E4%B8%AD%E6%89%BE%E5%88%B0+%28&input_invalid_notice_limits=+%3Cbr%2F%3E%E6%B3%A8%E6%84%8F%3A+%E6%BB%9A%E5%8A%A8%E6%A1%86%E4%B8%AD%E6%98%BE%E7%A4%BA%E7%9A%84%E5%AD%97%E6%AE%B5%E5%BF%85%E9%A1%BB%E8%87%B3%E5%B0%91%E4%B8%8E%E4%B8%80%E4%B8%AA%E5%85%B6%E4%BB%96%E6%A3%80%E7%B4%A2%E5%AD%97%E6%AE%B5%E7%9B%B8%E7%BB%84%E9%85%8D%E3%80%82&sa_params=WOS%7C%7C8CObDVoqT2Y2FTJnkg5%7Chttp%3A%2F%2Fapps.webofknowledge.com%7C%27&formUpdated=true&value%28select1%29=DO&value%28hidInput1%29=&limitStatus=collapsed&ss_lemmatization=On&ss_spellchecking=Suggest&SinceLastVisit_UTC=&SinceLastVisit_DATE=&period=Range+Selection&range=ALL&startYear=1977&endYear=2020&editions=SCI&editions=SSCI&editions=AHCI&editions=ESCI&editions=CCR&editions=IC&update_back2search_link_param=yes&ssStatus=display%3Anone&ss_showsuggestions=ON&ss_numDefaultGeneralSearchFields=1&ss_query_language=&rs_sort_by=PY.D%3BLD.D%3BSO.A%3BVL.D%3BPG.A%3BAU.A&value%28input1%29=";
			ThreadPoolExecutor executor = (ThreadPoolExecutor) Executors.newFixedThreadPool(8);
	        BlockingQueue<BasicPaper> que=new LinkedBlockingQueue<BasicPaper>();
	        if(StringUtil.isNotBlank(sid.get())) {
				for (Paper paper : paperList) {
					/*String[] sourceUrls = StringUtils.split(paper.getUrl(),"	");
					for(String sourceUrl : sourceUrls){*/
						if(paper.getUrl().contains("apps.webofknowledge.com")){
							//System.out.println(paper.getName());
							//String[] urls = StringUtils.split(sourceUrl,"\\~");
							
							executor.execute(new JcrSearchThread(cb,paper.getName(),url+paper.getDoi()+"&SID=",sid,que));
							
							/*Context2 = HttpClientUtil.getHttpEntityByGet(urls[1].replaceAll("&nbsp;", " ")+SID);
							Document docDetail=Jsoup.parse(Context2);
					        String journalInfo3=docDetail.getElementsByClass("search-results-data-cite").text();
					        String city=journalInfo3.split(":")[1].split("\\(")[0].trim();
					        System.out.println(city);*/
						}
					
				}
	        }
			System.out.println(" 队列长度："+executor.getQueue().size());
			executor.shutdown();
	        /*while(true){
	        	//监听线程池的所有任务完成
				if(executor.isTerminated()){
		        	new Thread(new JcrSearchThread(que)).start();
		        	break;
		        }
			}*/
		}

		//每爬取八篇论文就进行判断，如果够60s了就发送请求来维持sid，其实60s只是一种猜测，
		//后来使用同一篇论文循环请求测试，大概140篇左右sid就失效了，因此这里可以修改为每爬取100篇论文就重新请求sid。
		public CyclicBarrier cb=new CyclicBarrier(8,new Runnable(){
	        public void run(){
	        	long end = System.currentTimeMillis();
	            if((end - start)>60000) {
	            	System.out.println("更换SID");
	            	try {
						sid.set(getSID());
						System.out.println("新的SID:"+sid.get());
					} catch (NullPointerException e) {
						// TODO Auto-generated catch block
						e.printStackTrace();
					} catch (IOException e) {
						// TODO Auto-generated catch block
						e.printStackTrace();
					}
	            	start=System.currentTimeMillis();
	            }
	            
	        }
	    });
		
		public  static String getHttpEntityByGet(String url)throws Exception {
	    	WebClient webClient = new WebClient(BrowserVersion.CHROME);
	
	    	//设置协议，单个的
			webClient.getOptions().setUseInsecureSSL(true);		
			webClient.getOptions().setSSLInsecureProtocol("TLSv1.2");
			
			webClient.getOptions().setCssEnabled(false);
			webClient.getOptions().setJavaScriptEnabled(false);
			HtmlPage page = webClient.getPage(url);
			webClient.close();
			return page.asXml();
		}
	}

	//Package B
	public class JcrSearchThread implements Runnable{
		private String name;
		private String link;
		private CyclicBarrier cb;
		private AtomicReference<String> sid;
		private BlockingQueue<BasicPaper> que;
		private AtomicInteger integer;

		public JcrSearchThread(CyclicBarrier cb,String name,String link,AtomicReference<String> sid,BlockingQueue<BasicPaper> que) {
			this.name = name;
			this.cb = cb;
			this.link = link;
			this.sid = sid;
			this.que = que;
		}

		@Override
		public void run() {
			 String journalInfo3="";
			try {
		        String Context = JcrTest.getHttpEntityByGet(link+sid.get());
		        Document docDetail=Jsoup.parse(Context);
		        journalInfo3=docDetail.getElementsByClass("search-results-data-cite").text();
		        if(StringUtil.isNotBlank(journalInfo3)) {
		        String city=journalInfo3.split(":")[1].split("\\(")[0].trim();
		        System.out.println(name);
		        System.out.println(city);
				//在这里进行等待
		        cb.await();
		        }
		        //cb.reset();
			} catch(ArrayIndexOutOfBoundsException e){
				System.out.println(link);
				System.out.println("问题的论文名："+name);
				System.out.println("节点："+journalInfo3);
				
			}
			catch (Exception e) {
				e.printStackTrace();
			}
		}
	}

	//Package others
	protected DefaultHttpClient httpClient = new DefaultHttpClient();
	public String getSID() throws NullPointerException, IOException {
		String sid = "";
		// 发送请求
		executeGetMethod(SID_URL, false, true);
		// 获取Cookie中SID的值
		List<Cookie> cookies = httpClient.getCookieStore().getCookies();
		for (int i = 0; i < cookies.size(); i++) {
			if ("SID".equals(cookies.get(i).getName())) {
				sid = cookies.get(i).getValue().trim();
				break;
			}
		}
		return sid;
	}
	protected HttpResponse executeGetMethod(String url, boolean isUri,boolean isConsume) {
		HttpResponse response = null;
		HttpGet getMethod = null;
		// isUri 是true 的时候讲url转化成uri
		if (isUri) {
			getMethod = new HttpGet(getURIByURL(url));
		} else {
			getMethod = new HttpGet(url);
		}
		try {
			// 发送get请求
			response = httpClient.execute(getMethod);
			if(isConsume){
				EntityUtils.consume(response.getEntity());
			}
		} catch (ClientProtocolException e) {
			logger.error("通过url 获取一个response请求,URL为："+url,e);
			e.printStackTrace();
		} catch (IOException e) {
			logger.error("通过url 获取一个response请求,URL为："+url,e);
			e.printStackTrace();
		}
		return response;
	}

代码内容分为三部分，第一部分主要是一些业务操作，第二部分是发请求去获取引频次数，第三部分是一些论文爬取用到的内容。
