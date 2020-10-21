
---
title: "Building forms with Django Rest Framework and Vue.js"
date: 2020-10-18T17:20:00+01:00
draft: false
---

In the early days of Nuabee, we wrote our entire site using Django, and we mixed in sprinkles of jQuery when we needed something slightly dynamic. As the requirements evolved, we needed to implement some more complex interfaces, and jQuery just wasn't cutting it.

Rather than rewriting our entire frontend with a framework like Vue.js or React, we decided that it would be best to integrate a framework only on the pages that we needed. The idea was to have the best of both worlds; the ease of development with Django and the dynamic nature of a frontend framework.

If you're familiar with Django forms, you're well aware of how easy it is to add a model and then create a view that allows you to create or update that data. Django takes care of retrieving the data, error handling, updating or creating new entries in the database. These excellent features make static pages a breeze to develop. But how you take advantage of Django when you're using a frontend framework?

This blog post will outline how you can replicate the features of Django forms and integrate them within a Vue.js frontend. I'm going to assume that you've already integrated Vue.js within your Django application and you want to know how to build a simple form. I will also be writing all the frontend code in Typescript.

If you're just after the finished code or you want to see how to integrate webpack into Django, you can take a look [here](https://github.com/mierz00/django-vuejs-hybrid).

We're going to go through the following steps:

1. Add a Django Rest Framework view.
2. Add a Serializer.
3. Develop the form Vue.js template.
    1. Add a generic input component
4. Retrieve the form options.
5. Retrieve any initial data if it's an update page.
6. Submit the form and handle errors.

I've gone ahead and created a new music app and I've filled it with the following files:

```
├── __init__.py
├── admin.py
├── apps.py
├── models.py
├── serializers.py
├── static
│   └── js
│       ├── album_form
│       │   ├── app.types.ts
│       │   ├── components
│       │   │   ├── AlbumForm.ts
│       │   │   └── input
│       │   │       ├── Input.ts
│       │   │       ├── InputErrors.ts
│       │   │       ├── InputLabel.ts
│       │   │       └── templates
│       │   │           ├── checkbox.ts
│       │   │           ├── index.ts
│       │   │           ├── select.ts
│       │   │           └── standard.ts
│       │   └── index.ts
│       └── utils
│           └── cookies.ts
├── templates
│   └── album_form.html
├── tests.py
├── urls.py
└── views.py
```

We're going to need a couple of models. I've taken an example from the Django documentation which will work well for us!

Within our `models.py` file we're going add the following models:

```python
from django.db import models

class Musician(models.Model):
    def __str__(self):
        return "{0} {1}".format(self.first_name, self.last_name)

    first_name = models.CharField(max_length=50)
    last_name = models.CharField(max_length=50)
    instrument = models.CharField(max_length=100)

class Album(models.Model):
    def __str__(self):
        return "{0} - {1}".format(self.name, self.release_date)

    artist = models.ForeignKey(Musician, on_delete=models.CASCADE)
    name = models.CharField(max_length=100)
    release_date = models.DateField()
    num_stars = models.IntegerField(verbose_name="Number of stars")

```

Once that's done we need to run a `makemigrations` and `migrate` to update our database with the tables.

### Views

For this example, we're going to be building a form that allows us to create and update an Album.

Now that we have the two models, we can get right into it. For the moment, we're going to add two generic [Django Rest Framework views](https://www.django-rest-framework.org/api-guide/generic-views/) within our `[views.py](http://views.py)` file.

```python
from django.views.generic import TemplateView
from rest_framework import generics
from apps.music.models import Album

class AlbumCreate(TemplateView):
    template_name = "album_form.html"

class AlbumUpdate(TemplateView):
    template_name = "album_form.html"

    def get(self, request, *args, **kwargs):
        album_id = self.kwargs.get("pk")

        try:
            Album.objects.get(id=album_id)
        except Album.DoesNotExist:
            raise Http404()

        context = {"album_id": album_id}
        return self.render_to_response(context)

class AlbumList(generics.ListCreateAPIView):
    queryset = Album.objects.all()
    serializer_class = AlbumSerializer
    metadata_class = OptionsMetaData

class AlbumDetail(generics.RetrieveUpdateDestroyAPIView):
    queryset = Album.objects.all()
    serializer_class = AlbumSerializer
    metadata_class = OptionsMetaData
```

If you're unfamiliar with class based views, Django is going to give us a ton of functionality with these two classes. We will have all the CRUD HTTP functions such as GET, POST, PUT and DELETE.

We're also going to need two URLs to link up these views inside `urls.py`

```python
from django.urls import path
from rest_framework.urlpatterns import format_suffix_patterns
from * import views

urlpatterns = [
    path('albums/', views.AlbumList.as_view()),
    path('albums/<int:pk>', views.AlbumDetail.as_view()),
]

urlpatterns = format_suffix_patterns(urlpatterns)
```

### Serializer

Now it's time to create our Album serializer. The serializer will allow Django to be able to easily handle the exchange of data between the frontend and our application.

We're going to keep things simple and just use a model serializer within `serializers.py`

```python
from rest_framework import generics
from apps.music.models import Album
from apps.music.serializers import AlbumSerializer
from django.utils.encoding import force_text
from rest_framework.metadata import SimpleMetadata
from rest_framework.relations import RelatedField, ManyRelatedField

class AlbumList(generics.ListCreateAPIView):
    queryset = Album.objects.all()
    serializer_class = AlbumSerializer
    metadata_class = OptionsMetaData

class AlbumDetail(generics.RetrieveUpdateDestroyAPIView):
    queryset = Album.objects.all()
    serializer_class = AlbumSerializer
    metadata_class = OptionsMetaData

class OptionsMetaData(SimpleMetadata):
    def get_field_info(self, field):
        field_info = super(OptionsMetaData, self).get_field_info(field)
        if isinstance(field, (RelatedField, ManyRelatedField)):
            field_info["choices"] = [
                {
                    "value": choice_value,
                    "display_name": force_text(choice_name, strings_only=True),
                }
                for choice_value, choice_name in field.get_choices().items()
            ]
        return field_info
```

The `OptionsMetaData` class is going to allow us to make OPTION HTTP requests to get all the form information.

To check that everything is working so far, I added a couple of musicians to the database:

![Test data](/django-vue-form/db.png)

We can now check that our rest views are working correctly if we head to `127.0.0.1/8000/music/albums`

![Album list test](/django-vue-form/album_list.png)

If we click OPTIONS we should get a list of all the options:

![Album list test](/django-vue-form/album_list_2.png)

### Developing the Vue.js form

The way my Django application is set up is that we can create many Vue.js applications, transpile them with Webpack and then import them into a Django template. This way you do this may vary, so I'm going to skip that part. 

I've used Django webpack loader so my template looks like this:

```html
{% load render_bundle from webpack_loader %}
{% render_bundle 'chunk-vendors' %}

<script>
    const album_id =  {% if album_id %}{{ album_id }}{% else %}undefined{% endif %};
</script>

<div id="app" v-cloak >
    <album-form/>
</div>

{% render_bundle 'album_form' %}
```

I've included the variable `album_id` so that if we're on the update page, we can get the existing data for an album and then use that id to send a request to update the album.

Let's create an entry point inside `static/js/album_form/index.ts`

```typescript
import Vue from 'vue';
import AlbumForm from './components/AlbumForm';

const app = new Vue({
    el: '#app',
    components: { AlbumForm },
});
```

We want to create our TypeScript types within `app.types.ts` where we have our Album interface, a field interface which is used to retrieve the options from Django Rest Framework and a AlbumFormFields that contains fields for our form.

```typescript
export interface Field {
    type: string;
    required: boolean;
    read_only: boolean;
    label: string;
    choices?: Array<FieldChoice>;
}

export interface FieldChoice {
    value: number;
    display_name: string;
}

export interface AlbumFormFields {
    artist: Field,
    name: Field,
    releaseDate: Field,
    numStars: Field
}

export interface Album {
    artist: number;
    name: String;
    releaseDate: string;
    numStars: number;
}

export interface AlbumErrors {
    artist?: Array<string>;
    name?: Array<string>;
    releaseDate?: Array<string>;
    numStars?: Array<string>;
}
```

Next we want to develop the basis of our form. The idea is to have something that flows as follows: 

1. Load the component.
2. Get all the options from the Django Rest API.
3. Initialize any data.
4. Have the formData object update as we modify any inputs.
5. Submit the formData to the server.
6. Display any errors or redirect to the next page.

Inside our `components/AlbumForm.ts`

```typescript
import Vue from 'vue';
import { Component } from 'vue-property-decorator';
import Input from './input/Input';
import { Album, AlbumErrors, AlbumFormFields } from '../app.types';
import axios from 'axios';

const template = `
    <div v-if="loading" >
        Loading...
    </div>
    <form 
        v-else 
        @submit="submitForm" >   
            <Input 
                v-for="(field, fieldName) in fields"
                :key="fieldName"
                :name="fieldName"
                :initial="formData[fieldName]"
                :field="field"
                :errors="errors[fieldName]"
                @on-change="onChangeField"
            />
            <div>
              <button type="submit">
                Submit
              </button>
            </div>
    </form>
`;

@Component({
    template,
    name: 'AlbumForm',
    components: { Input },
})
export default class AlbumForm extends Vue {

    formData: Album = <Album>{};
    fields: AlbumFormFields = <AlbumFormFields>{};
    errors: AlbumErrors = {};
    loading: boolean = true;

    mounted() {
        // Initialise our form data.
        // This data is what we will send to the server
        this.formData = {
            artist: 0,
            name: 'test',
            releaseDate: '',
            numStars: 0,
        };

        this.getFormOptions();
        this.getAlbumData();
    }

    getAlbumData() {
		  // TODO - get album data if the page is update
    }

    getFormOptions() {
			// TODO - add get form
    }

    submitForm(e: Event) {
			 // TODO - submit form
        e.preventDefault();
    }

    // Handle the on change of any input
    onChangeField(field: string, value: any) {
        // @ts-ignore - so that we can use field inside formData
        this.formData[field] = value;
    }
}
```

Here there a quite a few things going on, we have created a AlbumForm class, and we've added a form to our template. Inside the form we display every Input which is in the fields. 

For the moment, we haven't written `getAlbumData()`, `getFormOptions()` and the `submitForm()` functions. We're going to gloss over that for a second and write the code for our generic Input.

### Generic input component for Django forms

I wanted to have a Vue.js component that would be able to display the standard inputs, like a checkbox, a selector or just a text input. To do this I used the following setup within an `input` folder :

```
├── components
   ├── AlbumForm.ts
   └── input
       ├── Input.ts
       ├── InputErrors.ts
       ├── InputLabel.ts
       └── templates
           ├── checkbox.ts
           ├── index.ts
           ├── select.ts
           └── standard.ts
```

Inside our `Input.ts` file we have: 

```typescript
import Vue from 'vue';
import { Component, Prop, Watch } from 'vue-property-decorator';
import { Field } from '../../app.types';
import templates from "./templates";
import InputLabel from './InputLabel';
import InputErrors from './InputErrors';

@Component({
    name: 'Input',
    components: {InputLabel, InputErrors}
})
export default class Input extends Vue {
    @Prop()
    name!: string;

    @Prop()
    field!: Field;

    @Prop()
    initial!: any;

    @Prop()
    errors?: [];

    @Prop({ default: false })
    multiSelect!: boolean;

    // DATA
    binding: any = '';

    // COMPUTED
    get InputTemplate() {
        switch(this.fieldType) {
            case 'checkbox':
                return templates.checkbox;
            case 'field':
                return  templates.select;
            default:
                return templates.standard;
        }

    }

    get fieldType() {
        switch (this.field.type) {
            case 'string':
                return 'text';
            case 'integer':
                return 'number';
            case 'boolean':
                return 'checkbox';
            default:
                return this.field.type;
        }
    }

    @Watch('binding')
    onChangeValue(value: any) {
        this.$emit('on-change', this.name, value);
    }

    // METHODS
    created() {
        this.binding = this.initial;
        this.$options.template = this.InputTemplate;
    }

}
```

This component will allow us to render any type of Django input. Depending on what the `field` attribute is, we will display a different template. 

I've made three templates:

1. Checkbox
2. Standard (numbers, text, dates) 
3. Select (single select or multiple select)

`checkbox.ts`

```typescript
export const checkbox = `
        <div>
            <div>
                <input
                    :name="name"
                    :type="fieldType"
                    v-model="binding"
                />
                <InputLabel :label="field.label" :required="field.required" />
                <InputErrors :errors="errors" />
            </div>
        </div>
`;
```

`standard.ts`

```tsx
export const standard = `
        <div>
            <InputLabel :label="field.label" :required="field.required" />
            <input
                :name="name"
                :type="fieldType"
                v-model="binding"
                :required="field.required"
            />
            <InputErrors :errors="errors" />
        </div>
`;
```

`select.ts`

```tsx
export const select = `
        <div>
            <InputLabel :label="field.label" :required="field.required" />
            <select
                :name="name"
                :multiple="multiSelect"
                :required="field.required"
                v-model="binding"
            >
                <option v-for="choice in field.choices" :value="choice.value">
                    {{ choice.display_name }}
                </option>
            </select>
            <InputErrors :errors="errors" />
        </div>
`;
```

You may have noticed, there are two components within these templates. `InputLabel` and `InputErrors`. I've added these just to simplify the code a little bit. 

`InputLabel.ts`

```typescript
import Vue from "vue";
import { Component, Prop } from 'vue-property-decorator';

@Component({
    name: "InputLabel",
    template: `
        <label>{{ label }}<span v-if="required">*</span></label>
    `
})
export default class InputLabel extends Vue {
    @Prop()
    label!: string;

    @Prop({default: false})
    required!: boolean;
}
```

`InputErrors.ts`

```typescript
import Vue from "vue";
import { Component, Prop } from 'vue-property-decorator';

@Component({
    name: "InputErrors",
    template: `
        <div v-if="errors">
            <p v-for="error in errors">
                {{ error }}
            </p>
        </div>
    `
})
export default class InputErrors extends Vue {
    @Prop()
    errors!: Array<string>;
}
```

### Getting the CSRF Token

We've set up our interface and the components, but now we need a way to communicate with the Django Rest API.

When using making POST or PUT requests to Django, you normally need to have a CSRF token. In our case, we can get this token from the cookies of the page. To do this, I added `utils/cookies.ts`:

```typescript
export function getCookie(c_name: string): string {
    if (document.cookie.length > 0) {
        let c_start = document.cookie.indexOf(c_name + '=');
        if (c_start != -1) {
            c_start = c_start + c_name.length + 1;
            let c_end = document.cookie.indexOf(';', c_start);
            if (c_end == -1) c_end = document.cookie.length;
            return unescape(document.cookie.substring(c_start, c_end));
        }
    }
    return '';
};
```

And we can add this to all of our request headers when we use `axios` module.

`AlbumForm.ts`

```typescript
import Vue from 'vue';
import { Component } from 'vue-property-decorator';
import Input from './input/Input';
import { Album, AlbumErrors, AlbumFormFields } from '../app.types';
import axios from 'axios';
import { getCookie } from '../../utils/cookies';

const Axios = axios.create({
    baseURL: '/music',
    headers: { 'X-CSRFToken': getCookie('csrftoken') },
});

...
```

Now any of our requests will contain our CSRF token.

### Getting the field options

To get all the field options, we take advantage of the Django Rest Framework metadata. By sending an OPTION request to the API, we can retrieve all the options and choices. Including foreign key fields. 

Let's fill out our `getFormOptions()` function within `AlbumForm.ts` :

```typescript
getFormOptions() {
        Axios.options('/api/albums')
            .then((res) => {
                this.fields = res.data.actions.POST;
            })
            .catch((err) => {
                console.log('err', err);
            })
            .finally(() => {
                this.loading = false;
            });
    }
```

We should now be able to load the page and see our fields with the different forms and options:

![form](/django-vue-form/form.png)

If we click on the Artist, we should see :

![Form select](/django-vue-form/form_2.png)

### Submit form function

Before we submit our form we want to handle two different cases, if it's an update or if it's a create page. If it's an update page we want to submit a PUT request to `/music/api/albums/<int:pk>`. This means that we need to use the `album_id` which was sent to the page using the `get_context` within the update view.

We want to update our data and `mounted()` function within `AlbumForm.ts`. This way we can access the album_id within any of our Vue.js functions.

```typescript
export default class AlbumForm extends Vue {

    album_id: number | undefined; // ADD THIS
    formData: Album = <Album>{};
    fields: AlbumFormFields = <AlbumFormFields>{};
    errors: AlbumErrors = {};
    loading: boolean = true;

    mounted() {
        // @ts-ignore - Comes from the Django template
        this.album_id = album_id;
        
        // Initialise our form data.
        // This data is what we will send to the server
        this.formData = {
            artist: 0,
            name: 'test',
            releaseDate: '',
            numStars: 0,
        };

        this.getFormOptions();
        this.getAlbumData();
    }
```

The `submitForm()` function is relatively simple, if we're on the update page it should send a PUT request to the API, otherwise it will send a POST request with the form data.

```typescript
submitForm(e: Event) {
    if (this.album_id) {
        Axios.put(`/api/albums/${this.album_id}`, this.formData)
            .then(this.handleSuccessSubmit)
            .catch(this.handleErrorSubmit);
    } else {
        Axios.post(`/api/albums/`, this.formData)
            .then(this.handleSuccessSubmit)
            .catch(this.handleErrorSubmit);
    }

    e.preventDefault();
}

handleSuccessSubmit(res: any) {
        console.log(res);
  }

handleErrorSubmit(err: any) {
    this.errors = err.response.data;
}
```

To finish it up, I've added some styles with Tailwind CSS :)

![Screenshot with Tailwind](/django-vue-form/screenshot.png)