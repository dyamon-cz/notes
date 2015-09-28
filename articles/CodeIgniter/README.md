# Jak jsem zkoušel CodeIgniter - CI <br>

Kamarád mě požádal o nějaké názory na PHP frameworky, předmětem jeho zájmu bylo hlavně [Nette](https://nette.org/), [Symfony](https://symfony.com/) a [CodeIgniter](http://www.codeigniter.com/). V Nette pracuji už nějaký ten pátek, mám i nějaké zkušenosti se Symfony, ovšem  o CodeIgniteru jsem nikdy neslyšel. Rozhodl jsem se na něj trochu podívat.<br>
Jako první jsem zamířil na [SitePoint](http://www.sitepoint.com/best-php-framework-2015-sitepoint-survey-results/), kde se hodnotila popularita PHP frameworků za rok 2015 a hle, CodeIgniter se umístil na 4. místě hned po Nette. Říkám si že to bude asi zajímavý a rovnocenný boj… zmýlil jsem se, nicméně zajímavé to bylo.

*Jednotlivé funkcionality CodeIgniteru hodnotím především z hlediska uživatele Nette Frameworku.*

##Absence objektové architektury

Abych uvedl nadpis na pravou míru, tak třídy tu jsou, nicméně o objektovém návrhu nemůže být ani řeč.<br> 
Základními kameny aplikace pro vás budou třídy [CI_Controller](http://www.codeigniter.com/user_guide/general/controllers.html) a [CI_Model](http://www.codeigniter.com/user_guide/general/models.html). Pokud se podíváte do [implementace](https://github.com/bcit-ci/CodeIgniter/blob/develop/system/core/Controller.php) těchto tříd, zjistíte že nemají žádné metody. Pomocí jakési magie mapují funkce, nadefinované v globálním prostoru a to má prakticky jenom samé nevýhody. PHPStorm neustále křičí, že voláte neznámé metody a používáte neznáme property (možná to vyřeší plugin, ale proč?), každopádně se vraťte klidně k textovým editorům, protože využití IDE tu je hodně omezené.<br>
V pozdější fázi budete donuceni sami používat funkce z globálního scopu, akorát se jim tu říká [helpery](http://www.codeigniter.com/user_guide/helpers/index.html) a můžete si je postupně zpřístupnit.<br>
Framework **nepoužívá namespaces** a nenajdete tu ani žádný [Robot Loader](https://doc.nette.org/cs/2.3/auto-loading), místo toho dodržujete konvence, controllery do složky **controllers**, modelové třídy do složky **models** a tak dále. Pokud byste ovšem chtěli použít operátor new, musíte si třídu stejně připojit ručně pomocí require. Vzápětí zjistíte proč.

##Získávání závislostí

[Singletony](https://sites.google.com/site/steveyegge2/singleton-considered-stupid), [singletony](http://www.c2.com/cgi/wiki?SingletonsAreEvil), [singletony](https://phpfashion.com/je-singleton-zlo)… desítky článků o tomto antipatternu a CodeIgniter si ho vesele používá dál. Závislosti můžete získávat, když podědíte od **CI_Controller** nebo **CI_Model**. V neviditelné magické property `$load` se nachází [CI_Loader](http://www.codeigniter.com/user_guide/libraries/loader.html).

```php
$this->load->view('view.php');
$this->load->model('news_model');
```

Místo názvu třídy s namespace mu předáváte “pseudocestu” ke třídě. Takže pokud budete mít modelovou třídu **News** v souboru **News.php** a ve složce **models/data**, načte vám jí, pokud zavoláte `$this->load->model('data/news')`. V případě, že vám adresářová struktura později nebude vyhovovat, víte co vás <em title="přepisování cest, všude kde načítáte závislosti">čeká<sup>?</sup></em>. Navíc si zvykněte na používání prefixů/sufixů.<br>
Dále dělá **CI_Loader** opravdu hodně věcí. Když načtete **view**, tak vám ho rovnou vyrenderuje, u modelových tříd a tzv. [libraries](http://www.codeigniter.com/user_guide/libraries/index.html), vám zpřístupní podle nich pojmenované property.

```php
$this->load->model('news_model');
$this->news_model->findAllNews();
```

A nakonec to nejlepší - po načtení helperu budete moci začít používat nové funkce z globálního prostoru [# :) #](http://oi60.tinypic.com/2la5wk0.jpg) (fu\*k you [SRP](https://en.wikipedia.org/wiki/Single_responsibility_principle), fu\*k you OOP).<br>
Víme, jak skvělé jsou neviditelné závislosti, takže nám tohle všechno samozřejmě hodně opepří nějaké pokusy o testování, nebo refactoring. Vlastně si myslím, že kdybych se k takové aplikaci vrátil například po roce, budu mít dost velké problémy se zorientovat co na čem závisí a proč.

##Konfigurace

Po vytvoření vlastního php souboru ve složce **config**, můžete přidávat svá konfigurační data do předepsaného pole `$config` v globálním scopu.<br>
Pro přístup k těmto datům pokračujete opět ve stejném duchu. Podědíte **CI_Controller** nebo **CI_Model** a sáhnete si na magickou proměnou `$config`. V té se nachází třída [CI_Config](http://www.codeigniter.com/user_guide/libraries/config.html), pomocí níž můžete s veškerou konfigurací manipulovat. Ale pozor - vlastní soubor si nejdříve musíte načíst `$this->config->load('my_config.php')` (jakoby už to nebylo úplně jedno).<br>
K neviditelným závislostem se nám tak přidává další kurvítko. Nemusíte mít ani žádnou rozsáhlou business logiku, aby nastal problém. Jestliže necháte cokoli záviset na takové konfiguraci, nikdo vám nezaručí, že vám jí někdo někde nepřepíše.

##Routování

Pokud nemá url vyhrazenou routu, bude mapován stylem **[controller]/[method]/[arguments]**. Vlastní routy si definujete v konfiguračním php souboru. [Syntaxe](http://www.codeigniter.com/user_guide/general/routing.html) je jednoduchá a snadno pochopitelná, až se z počátku zdálo, že to funguje docela hezky.<br> 
Hned po nastavení výchozí routy se ovšem objevil první problém. Fungují totiž všechny tři url adresy. Framework vám naservíruje obsah stránky, když vstoupíte na routu defaultního controlleru, stejně tak jako když použijete **localhost/welcome**, nebo **localhost/welcome/index**.<br>
Problém spočívá v tom, že se stejný obsah nachází na více url adresách. Toto je tak <a href="http://vyhledavace.info/seo-faq/7/duplicitni-obsah" title="věnujte pozornost datu zveřejnění">staré<sup>?<sup></a> téma, že jsem se dost divil absenci této fičury ([nette](https://doc.nette.org/cs/2.3/routing#toc-seo-a-kanonizace) jí má co si ho pamatuji). Samozřejmě si můžete tento problém vyřešit sami, [zde](http://www.farinspace.com/codeigniter-htaccess-file/) například pomocí .htaccess (nebo jiného konfiguračního souboru, dle vašeho používaného serveru), přeji hodně [štěstí](https://phpfashion.com/vite-komu-ublizil-mod_rewrite).<br>
Další komfort na který můžete zapomenout je, že by vám například router, podle slugu, naservíroval rovnou entitu, se kterou chcete pracovat.<br>
Také jsem hledal podobou [fičuru](https://doc.nette.org/cs/2.3/presenters#toc-vytvareni-odkazu), která zajistí, aby byly odkazy nezávislé na nastavení rout. Trochu jsem doufal, že by to mohl řešit [URL helper](http://www.codeigniter.com/user_guide/helpers/url_helper.html), ale kdepak, umí pouze přidávat base path a jinak tupě odpapouškuje co mu napíšete. Pokud se tedy u hotové aplikace rozhodnete změnit routy, čeká vás nepříjemné přepisování veškerých odkazů v aplikaci.

##Formuláře

V Nette jsou formuláře jednou z mých vůbec nejoblíbenějších fičur a dost mě zajímalo, jak na to jde CodeIgniter. V dokumentaci na ně poprvé narazíte v tutoriálu [Create news items](http://www.codeigniter.com/user_guide/tutorial/create_news_items.html).<br>
Na generování formulářových prvků má CodeIgniter opět [helper](http://www.codeigniter.com/user_guide/helpers/form_helper.html), který vám zpřístupní takové funkce jako `form_open`, `form_input`, `form_password` atp. Tyto funkce voláte na potřebném místě přímo šabloně.<br>
Na validace si načtete knihovnu [Form Validation](http://www.codeigniter.com/user_guide/libraries/form_validation.html) a tu zase používáte v metodě, na kterou nasměrujete formulářovou akci.

```php
$this->form_validation->set_rules('mail', 'Mail', 'required|valid_email');
```

Tyto validace se ve vzduchu nějak spojí s formulářem (<em title="neověřeno">předpokládám, že se podle předaného názvu prostě validují přímo data z POST<sup>?</sup></em>). Také se vám zpřístupní obecná funkce `validation_errors`, kterou si můžete v šabloně vypsat, aby vám renderovala porušené validace.

```php
<?= validation_errors(); ?>

<?= form_open('welcome/process'); ?>
<?= form_label('Email:'); ?>
<?= form_input(array('name' => 'mail', 'type' => 'email')); ?>
<?= form_label('Password:'); ?>
<?= form_password(array('name' => 'password')); ?>
<?= form_submit(array('id' => 'submit', 'value' => 'Submit')); ?>
<?= form_close(); ?>
```

Každopádně v metodě, na kterou nasměrujete formulářovou akci, nesmíte zapomenout kontrolovat, jestli validace prošly... a vůbec tu nesmíte zapomenout na hodně věcí abyste zajistili nějakou tu bezpečnost.<br>
Myslím si, že ani v pozdější fázi používání této “knihovny”, si tu bez dokumentace neškrtnete. O nějakých komponentách tu samozřejmě vůbec nemůže být řeč, natož o znovupoužitelnosti vytvořených formulářů.

Když tohle srovnám s [Nette\Forms](https://doc.nette.org/cs/2.3/forms), kde je formulář prostě objekt, navíc [komponenta](https://doc.nette.org/cs/2.3/components), pro který vytvoříte továrničku, navěsíte na něj callback (volající se, pouze pokud projde validace) a naprosto [geniálním způsobem](https://phpfashion.com/formulare-v-nette-2-1#toc-nove-vykreslovaci-zbrane) si ho v šabloně necháte vyrenderovat… vlastně to mluví samo za sebe :).

##Dokumentace

[Dokumentaci](http://www.codeigniter.com/user_guide/index.html) jsem tu chtěl zvlášť zmínit, protože bez ní, a toho v jakém stavu je, by podle mého názoru, CodeIgniter vůbec nikdo nepoužíval. Jedná se o rozsáhlý a podrobný popis frameworku, včetně návodů jak se vypořádat s BC breaky, kterých je při této architektuře opravdu [požehnaně](http://www.codeigniter.com/user_guide/installation/upgrading.html). Díky dokumentaci se framework kdokoli hodně rychle naučí, včetně veškerých špatných návyků, které prezentuje.

*Možná by se mohli autoři méně soustředit na dokumentaci a více na to, že jejich framework zaspal dobu.*

##Nezaspěte s CodeIgniterem

Framework není jen [zastaralý](http://stackoverflow.com/questions/22645128/start-a-project-with-codeigniter#answer-22646433) a bez důležitých funkcionalit, ale hlavně nabádá k používání různých [bad practice](https://thecelavi.wordpress.com/2014/01/21/stay-away-from-codeigniter/), kde si hlavně začátečníci mohou odvést řadu špatných návyků. 
Proč je tedy tak populární? Možná, krom dokumentace, právě pro to, že nemá objektovou architekturu. Objektově orientované programování, je prostě náročnější než strukturované. A nenechte se zmást, to že se v kódu nachází třídy ještě [neznamená](http://stackoverflow.com/questions/18124553/my-first-oop-aproach-in-codeigniter), že je objektový.

Na začátku jsem zapomněl zmínit, že framework nevyžaduje šablonovací systém - v dokumentaci se dozvíte <a href="https://codeigniter.com/user_guide/overview/at_a_glance.html#codeigniter-does-not-require-a-template-engine" title="doporučuji přečíst :)">proč<sup>?<sup></a>. Také mi scházelo spoustu dalších užitečných pomůcek, hlavně [Tracy](https://tracy.nette.org/) a její [debug bar](https://tracy.nette.org/#toc-debugger-bar), kde pravidelně kontroluji, jak rychle se web načítá, v jakém jsem zrovna presenteru (controlleru), která routa je aktivní a spoustu dalších užitečných věcí. [Databázový layer](http://www.codeigniter.com/user_guide/database/query_builder.html) tu sice je, ale takové otřesné api bych déle, než za účelem tohoto testování, nedokázal používat.

*Jistě, spoustu těchto věci bych se mohl pokusit do CodeIgniteru zakomponovat, ale proč bych to dělal?*

Věřím, že jsem přehlédnul a nevyzkoušel spoustu věcí, které CI dále nabízí, problém je, že i kdyby některá z nich plnila svojí úlohu sebedokonaleji, nevyváží to jeho celkový přístup k vývoji aplikací. V anketě se sice umístil těsně na 4. místě, ale do Nette má tento framework opravdu hodně daleko.

Na závěr bych všem italským gurmánům, kteří stále používají CI doporučil, aby nezaspali (jako jejich framework) a přešli od [špaget](https://cs.wikipedia.org/wiki/%C5%A0pagetov%C3%BD_k%C3%B3d) k [OOP](https://nette.org/).