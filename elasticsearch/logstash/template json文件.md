{

"index_patterns": [

"*land_mass*"

],

"priority": 1,

"template": {

"mappings": {

"properties": {

"name": {

"type": "text",

"index": true,

"analyzer": "dic_ansj",

"search_analyzer": "dic_ansj"

},

"whole_address": {

"type": "text",

"index": true,

"analyzer": "dic_ansj",

"search_analyzer": "dic_ansj"

},

"remark": {

"type": "text",

"index": true,

"analyzer": "dic_ansj",

"search_analyzer": "dic_ansj"

},

"start_time_remark": {

"type": "text",

"index": true,

"analyzer": "dic_ansj",

"search_analyzer": "dic_ansj"

},

"end_time_remark": {

"type": "text",

"index": true,

"analyzer": "dic_ansj",

"search_analyzer": "dic_ansj"

},

"pitch_row": {

"type": "text",

"index": true,

"analyzer": "dic_ansj",

"search_analyzer": "dic_ansj"

},

"stage": {

"type": "keyword",

"ignore_above": 256

},

"owner": {

"type": "text",

"index": true,

"analyzer": "dic_ansj",

"search_analyzer": "dic_ansj",

"fields": {

"keyword": {

"type": "keyword",

"ignore_above": 256

}

}

},

"owner_name": {

"type": "keyword",

"ignore_above": 256

},

"owner_phone_number": {

"type": "keyword",

"ignore_above": 256

},

"exploration": {

"type": "text",

"index": true,

"analyzer": "dic_ansj",

"search_analyzer": "dic_ansj",

"fields": {

"keyword": {

"type": "keyword",

"ignore_above": 256

}

}

},

"exploration_name": {

"type": "keyword",

"ignore_above": 256

},

"exploration_phone_number": {

"type": "keyword",

"ignore_above": 256

},

"exploration_team": {

"type": "text",

"index": true,

"analyzer": "dic_ansj",

"search_analyzer": "dic_ansj",

"fields": {

"keyword": {

"type": "keyword",

"ignore_above": 256

}

}

},

"area": {

"type": "scaled_float",

"scaling_factor": 100,

"index": false

},

"longitude": {

"type": "double",

"index": false

},

"latitude": {

"type": "double",

"index": false

},

"id": {

"type": "long",

"index": false

}

}

}

}

}