<page actionBarHidden="true">     
    <tabs tabsPosition="bottom" bind:selectedIndex="{current_tab_idx}" on:selectedIndexChanged={tabs_index_change}>
        <tabStrip>
            <tabStripItem>
                <image src="font://&#xf03a;" class="fas" stretch="none"></image>
            </tabStripItem>
            <tabStripItem>
                <image src="font://&#xf1c0;" class="fas" stretch="none"></image>
            </tabStripItem>
            <tabStripItem>
                <image src="font://&#xf4fe;" class="fas" stretch="none"></image>
            </tabStripItem>
        </tabStrip>

        <tabContentItem class="tab_content"> <!-- Diario -->
            <stackLayout orientation="vertical" class="diario_stack">
            <gridLayout columns="*" rows="auto, auto, 2*, *">
                <gridLayout row="0" columns="*, 2*, *" rows="auto, auto" class="navbar">
                    <button row="0" col="0" text="&#xf060;" class="fas nav_button" on:tap="{date_previous}" width="150" height="30" />
                    <label bind:text="{current_date_string}" row="0" col="1" class="date_label" horizontalAlignment="center" verticalAlignment="center" />
                    <button row="0" col="2" text="&#xf061;" class="fas nav_button" on:tap="{date_next}" width="150" height="30" />
                    <label text="{daily_kcal} / {profile_day_kcal} Kcal" row="1" col="0" colspan="3" class="kcal_label" horizontalAlignment="center" verticalAlignment="center" />
                </gridLayout>
                <gridLayout row="1" columns="*, auto" rows="auto" style="margin-bottom:10;">
                    <textField row="0" col="0"  width="60%" bind:text="{diario_food_name}" hint="Alimento..." editable="true" returnKeyType="next" class="text_field" />
                    <textField row="0" col="1"  width="40%" bind:text="{diario_food_weight}" hint="Peso (g)..." editable="true" keyboardType="integer" returnKeyType="done" on:returnPress={diario_add} class="text_field"/>
                </gridLayout>
                
                <gridLayout row="2" columns="*" rows="*">
                <listView  items="{diario_list}" height="300" class="diario_list_view" on:itemTap={diario_list_tap}>
                    <Template let:item>
                         <stackLayout orientation="horizontal">
                            <label text="{item.name}" width="60%"/>
                            <label text="{item.kcal} KCal" width="40%" />
                        </stackLayout>
                    </Template>
                </listView>
                <activityIndicator busy={activity_busy_indicator} color="red"  width="40" height="40"></activityIndicator>
                </gridLayout>

                <gridLayout row="3" columns="auto, *" rows="auto">
                    <listPicker col="0"row="0" items="{alphabet_picker}" class="food_picker" selectedIndex="0" on:selectedIndexChange={alphabet_picker_change}/>
                    <listPicker col="1"row="0" items="{food_list_picker_filtered}" class="food_picker" selectedIndex="0" on:selectedIndexChange={food_picker_change} textField="name" valueField="kcal"/>
                </gridLayout>
            </stackLayout>
        </tabContentItem>

        <tabContentItem class="tab_content"> <!-- Alimenti -->
            <stackLayout orientation="vertical" class="food_stack">
                <gridLayout columns="auto, *" rows="auto, auto, auto, auto, auto" >
                    <label text="Gestione Alimenti" row="0" col="0" colspan="2" class="title_label" />
                    <label text="Alimento:" row="1" col="0" class="food_label" />
                    <textField hint="Inserisci alimento" row="1" col="1" bind:text="{food_name}" returnKeyType="next" class="text_field"/>
                    <label text="KCal/100g:" row="2" col="0" class="food_label" />
                    <textField hint="Inserisci KCal" row="2" col="1" keyboardType="integer" bind:text="{food_kcal}" on:returnPress={food_add} class="text_field"/>
                    <button text="Aggiungi" row="3" col="0" colspan="2" class="alimenti_button" on:tap={food_add} />
                    <activityIndicator row="4" col="0" colspan="2" busy={activity_busy_indicator} color="red"  width="40" height="40"></activityIndicator>
                </gridLayout>
            </stackLayout>
        </tabContentItem>

        <tabContentItem class="tab_content"> <!-- Profilo -->
            <stackLayout orientation="vertical" class="food_stack">
                <gridLayout columns="auto, *" rows="auto, auto, auto, auto" >
                    <label text="Profilo Utente" row="0" col="0" colspan="2" class="title_label" />
                    <label text="Nome Utente" row="1" col="0" class="food_label" />
                    <textField hint="Inserisci nome utente" row="1" col="1" bind:text="{profile_username}" returnKeyType="next" class="text_field"/>
                    <label text="KCal giornaliere" row="2" col="0" class="food_label" />
                    <textField hint="Inserisci KCal giornaliere" row="2" col="1" bind:text="{profile_day_kcal}" keyboardType="integer" returnKeyType="done" on:returnPress={profile_update} class="text_field"/>
                    <button text="Aggiorna" row="3" col="0" colspan="2" class="alimenti_button" on:tap={profile_update}/>
                </gridLayout>
            </stackLayout>
        </tabContentItem>
    </tabs>
</page>

<script>
    import { onMount } from 'svelte';
    import { Template } from 'svelte-native/components'
    import { ios } from 'tns-core-modules/application';
    import * as nsutils from 'tns-core-modules/utils/utils';
    import { Feedback, FeedbackPosition, FeedbackType } from "nativescript-feedback";
    var FeedbackPlugin = require("nativescript-feedback");
    const appSettings = require("tns-core-modules/application-settings");
    
    const removeFromList = (list, item) => list.filter(t => t !== item);
    
    const DB_ALIMENTI = "FoodLogger_food_DB";
    const DAILY_USER_DIARY = "FoodLogger_diary_";

    var firebase = require("nativescript-plugin-firebase");
    var firebaseConfig = {
        url: "https://<YOUR PROJECT ID>.firebaseio.com",
        persist: true
    };
    var feedback_dialog = new FeedbackPlugin.Feedback();
    var color = require("color");

    let food_name = "";
    let food_kcal = "";
    let food_list = [];
    let alphabet_picker = Array.from([...Array(26).keys()], item => String.fromCharCode(item + 65));
    let food_list_picker_filtered = [];
    let diario_list = [];
    let diario_food_name  = "";
    let diario_food_weight  = "";
    let current_date = new Date();
    let current_date_string = "";
    update_date(current_date);
    let current_tab_idx = 0; 
    let daily_kcal = 0;
    let profile_day_kcal;
    let profile_username = "";

    //to be removed
    let last_tab_idx = 0;
    let activity_busy_indicator = false;

    $: {
        console.log(food_list);
    }

    async function food_add() {
        console.log("add alimenti start");
        activity_busy_indicator = true;
        if (!food_entry_valid()) {
            console.log("ERROR: food item not valid. All field must be filled in");
            show_error(1000, "Errore Inserimento", "Tutti i campi devono essere valorizzati");
        } else {
            var food_item = {id: Date.now(),
                             name: food_name, 
                             kcal: food_kcal,
                             type: "food"};
            try {
                var docname = food_item['name'].replace(/\s/g, '');
                var collname = DB_ALIMENTI;
                await food_add_item_db(collname, docname, food_item);
                food_name = "";
                food_kcal = "";
                food_list = [...food_list, food_item];
                show_success(1000, "Inserimento Alimento", "Alimento inserito correttamente");
            } catch(error) {
                console.log("Errore inserimento alimento");
                console.log(error);
                show_error(1000, "Inserimento Alimento", "Impossibile inserire l'alimento nel DB");
            }
        }
        activity_busy_indicator = end;
        console.log("add alimenti end");
    }

    function get_local_profile() {
        console.log("get_local_profile start");
        if (!appSettings.hasKey("profile_username") ||
            !appSettings.hasKey("profile_day_kcal")) {
            console.log("profil is not complete or missing");
            show_error(3000, "Errore Profilo", "Il profilo utente non è completo");
            return false;
        } else {
            profile_username = appSettings.getString("profile_username");
            profile_day_kcal = appSettings.getNumber("profile_day_kcal");
            console.log("profile_username = " + profile_username);
            console.log("profile_day_kcal = " + profile_day_kcal);
            return true;
        }
        console.log("get_local_profile end");
    }

    async function get_daily_diario(username) {
        console.log("Recupero diario del giorno");
        daily_kcal = 0;
        try {
            console.log(DAILY_USER_DIARY + username);
            await firebase.firestore.collection(DAILY_USER_DIARY + username)
                .where("type", "==", "diario")
                .where("user", "==", profile_username)
                .where("yy", "==", current_date.getFullYear())
                .where("mm", "==", current_date.getMonth() + 1)
                .where("dd", "==", current_date.getDate())
                .get()
                .then(querySnapshot => {
                    diario_list = [];
                    querySnapshot.forEach(doc => {
                        //diario_list.push(doc.data());
                        diario_list = [...diario_list, doc.data()];
                        daily_kcal += parseInt(doc.data().kcal);
                        console.log(doc.data());
                    })        
            });
        } catch(error) {
            console.log("Errore nel recuperare diario del giorno");
            console.log(error);   
        }
        console.log("DIARIO => " + diario_list.length);
        console.log(diario_list);
    }

    onMount(async () => { 
        console.log("onMount start");
        activity_busy_indicator = true;
        if (!get_local_profile()) {
            console.log("ERROR getting local profile");
            current_tab_idx = 2;
            last_tab_idx = 2;
        }

        try {
            console.log("init firestore DB");
            await firebase.init(firebaseConfig);
        } catch(error) {
            console.log("Errore initializzazione firestore DB");
            console.log(error);
        }
        console.log("Firestore DB correttamente inizializzato");

        console.log("Recupero lista alimenti");
        
        try {
            await firebase.firestore.collection(DB_ALIMENTI)
                .where("type", "==", "food")
                .get()
                .then(querySnapshot => {
                    querySnapshot.forEach(doc => {
                        food_list = [...food_list, doc.data()];
                    })        
            });
        } catch(error) {
            console.log("Errore nel recuperare lista alimenti");
            console.log(error);   
        }
 
        /* OBSERVER FOR NOTIFICATION OF SB CHANGES
        const observer =  firebase.firestore.collection('test')
            .onSnapshot(querySnapshot => {
                querySnapshot.docChanges().forEach(change => {
                  if (change.type === 'added') {
                    console.log('New city: ', change.doc.data());
                  }
                  if (change.type === 'modified') {
                    console.log('Modified city: ', change.doc.data());
                  }
                  if (change.type === 'removed') {
                    console.log('Removed city: ', change.doc.data());
                  }
                });
              });
        */

        console.log("ALIMENTI => " + food_list.length);
        console.log(food_list);
        //food_list_picker = food_list_transform(food_list);
        food_list_picker_filtered = food_list;

        //get the dayly log of food
        get_daily_diario(profile_username);
        console.log("ACESS DB END");

        activity_busy_indicator = false;
        console.log("onMount end");
     })

    function hide_keyboard() {
        if (ios) {
            ios.nativeApp.sendActionToFromForEvent('resignFirstResponder', null, null, null);
        } else {
            nsutils.ad.dismissSoftInput();
        }
    }

    function food_entry_valid() {
        return (food_name != "" &&
                food_name != " " &&
                food_kcal > 0);
    }

    function diario_entry_valid() {
        return (diario_food_name != "" &&
                diario_food_weight > 0 && 
                profile_username != "" &&
                profile_day_kcal != 0);
    }

    function show_error(delay, title, mex) {
        if (ios)
            alert(mex);
        else
            feedback_dialog.show({
                title: title,
                message: mex,
                duration: delay,
                type: FeedbackType.Error,
            });
    }

    function show_success(delay, title, mex) {
        if (!ios)
            feedback_dialog.success({
                title: title,
                message: mex,
                duration: delay,
                backgroundColor: new color.Color("#0077B6"),
            });
    }

    function food_add_item_db(collname, docname, food_item) {
        var food_db = firebase.firestore.collection(collname);
        food_db.doc(docname).set(food_item);
        console.log('Aggiunto alimento: ' + food_item['name'] + ' -- kcal: ' + food_item['kcal']);
    }

    function update_date(current_date) {
        current_date_string =  current_date.getDate() + "/" + (current_date.getMonth() + 1) + "/" +current_date.getFullYear();
    }
    
    function date_previous()  {
        console.log("date_previous()");
        current_date.setDate(current_date.getDate() - 1);
        update_date(current_date);
        //diario_list = [];
        get_daily_diario(profile_username);
    }

    function date_next(){
        console.log("date_next()");
        current_date.setDate(current_date.getDate() + 1);
        update_date(current_date);
        //diario_list = [];
        get_daily_diario(profile_username);  
    }

    function food_picker_change(args) {
        console.log("food_picker_change");
        console.log(args.object.selectedIndex);
        console.log(food_list_picker_filtered[args.object.selectedIndex].name);
        diario_food_name = food_list_picker_filtered[args.object.selectedIndex].name;
    }

    function alphabet_picker_change(args) {
        console.log(args.object.selectedIndex);
        console.log(alphabet_picker[args.object.selectedIndex]);
        food_list_picker_filtered = food_list.filter(t => t.name.startsWith(alphabet_picker[args.object.selectedIndex]));
        if (food_list_picker_filtered.length != 0) {
            console.log(food_list_picker_filtered);
            diario_food_name = food_list_picker_filtered[0].name;
        }
        else {
            console.log("food_list_picker_filtered empty");
            food_list_picker_filtered = [{"name": "Nessun Alimento", 
                                         "kcal": 0, 
                                         "id": 0,
                                         "type": "food"}];
        }
    }

    async function diario_add() {
        console.log("add diario start");
        activity_busy_indicator = true;
        if (!diario_entry_valid()) {
            console.log("ERROR: diario item not valid. All field must be filled in");
            show_error(1000, "Errore Inserimento", "Tutti i campi devono essere valorizzati");
        } else {
            var dd = current_date.getDate();
            var mm = current_date.getMonth() + 1;
            var yy = current_date.getFullYear();
            var food_kcal_100 = food_list.find((item) => {return item.name == diario_food_name}).kcal;
            var diario_food_kcal = Math.ceil((food_kcal_100 / 100) * diario_food_weight);

            var diario_item = {id: Date.now(),
                             user: profile_username,
                             name: diario_food_name, 
                             kcal: diario_food_kcal,
                             type: "diario",
                             dd: dd,
                             mm: mm,
                             yy: yy};
            console.log(diario_item);
            try {
                var docname = profile_username + "_" + diario_item.id;
                var collname = DAILY_USER_DIARY + profile_username;
                await food_add_item_db(collname, docname, diario_item);
                diario_food_weight = "";
                show_success(1000, "Inserimento Alimento", "Alimento inserito correttamente");
                diario_list = [...diario_list, diario_item];
                daily_kcal += parseInt(diario_item.kcal);
            } catch(error) {
                console.log("Errore inserimento alimento");
                console.log(error);
                show_error(1000, "Inserimento Alimento", "Impossibile inserire l'alimento nel DB");
            }
        }

        activity_busy_indicator = false;
        console.log("add diario start");
    }

    function profile_update() {
        console.log("profilo update start");
        console.log(profile_username);
        if (profile_username == "" || 
            typeof profile_day_kcal == "undefined" ||
            profile_day_kcal == 0 ) {
            console.log("ERROR: profile info must be compelted");
            show_error(1000, "Inserimento Profilo", "Tutti i campi del profilo devono essere valorizzati");
        } else {
            console.log("Adding " + profile_day_kcal + " to user profile");
            console.log("Adding " + profile_username + " to user profile");
            appSettings.setString("profile_username", profile_username);
            appSettings.setNumber("profile_day_kcal", parseInt(profile_day_kcal));
            show_success(1000, "Inserimento Profilo", "Il profilo è stato inserito correttamente");
        }
        hide_keyboard();
        console.log("profilo update end");
    }

    function tabs_index_change(args){
        last_tab_idx = current_tab_idx;
        current_tab_idx = args.object.selectedIndex;
        if (last_tab_idx == 2 &&
            current_tab_idx == 0) {
                console.log("Reloading daily diario list");
                get_daily_diario(profile_username);
            }
        console.log(current_tab_idx);
    }

    async function diario_list_tap(args) {
        console.log("diario_list_tap");
        let result = await action("Vuoi eliminare questo alimento?",
                                  "Cancel",
                                  [ "Elimina"]
                                );
        switch(result) {
            case "Cancel" || undefined: 
                console.log("Mantieni alimento");
            break;
            case "Elimina":
                console.log("elimina alimento dal diario");
                let item = diario_list[args.index];
                let doc_id;

                //get the document to delete
                try {
                    await firebase.firestore.collection(DAILY_USER_DIARY + profile_username)
                          .where("id", "==", item.id)
                          .get()
                          .then(querySnapshot => {
                            querySnapshot.forEach(doc => {
                            doc_id = doc.id;
                          })
                    });  
                    
                    console.log(doc_id);
                    const doc_to_delete = firebase.firestore.collection(DAILY_USER_DIARY + profile_username).doc(doc_id);
                    doc_to_delete.delete().then(() => {
                      console.log("alimento rimosso dal diario");
                    });
                    diario_list = removeFromList(diario_list, item);
                } catch(error) {
                    console.log("Errore inserimento alimento");
                    console.log(error);
                    show_error(1000, "Rimozione Alimento", "Impossibile rimuovere l'alimento dal diario");
            }
            break;
        }
        console.log(result);
    }

</script>

<style>
    .diario_stack {
        border-color: #007F5F;
        margin: 0;
        padding: 10;
    }

    .navbar {
        margin-top: 0;
        margin-bottom: 0;
        background-color: transparent;
    }

    .nav_button {
        background-color: transparent;
        border-color: transparent;
        color: #03045E;
        font-size: 32;
        font-weight: bold;
        z-index: 0;
    }

    .date_label {
        font-size: 24;
        font-weight: bold;
        color: #03045E;
    }

    .kcal_label {
        font-size: 20;
        font-weight: bold;
        color: #03045E;
    }

    .food_picker {
        color: #0077B6;
        vertical-align: middle;
    }

    .food_stack {
        border-color: #007F5F;
        border-width: 0;
        border-radius: 30;
        margin: 0;
        padding: 5;
    }

    .food_label {
        color: #0077B6;
        vertical-alignment: center;
        horizontal-alignment: right;
        background-color: transparent;
        padding: 5;
        margin: 10 10 10 10;
        font-weight: bold;
    }

    .alimenti_button {
        background-color: #03045E;
        color: white;
        margin-top: 50;
        border-radius: 10;
        font-size: 24;
        font-weight: bold;
        border-color: #FBAC4B;
        border-width: 0;
    }

    .title_label {
        color: #03045E;
        border-color: yellow;
        border-width: 0;
        vertical-alignment: center;
        horizontal-alignment: center;
        font-size: 28;
        font-weight: bold;
        margin-bottom: 50;
    }

    TabStrip {
        selected-item-color: white;
        un-selected-item-color: #0077B6;
        highlight-color: #CAF0F8;
        background-color: #03045E;
    }  

    .tab_content {
        background-color: white;
    }

    .text_field {
        color: #0077B6;
        border-color: #03045E;
        border-width: 0 0 2 0;
        placeholder-color: #0077B6;
    }

    .diario_list_view {
        background-color: white;
        color: #03045E;
        separator-color: #03045E;
    }

</style>
