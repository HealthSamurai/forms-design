
# Table of Contents

1.  [Goal](#org07d7d94)
2.  [Decisions](#org3079260)
    1.  [**Remove :keys on top level**](#org2fb4829)
    2.  [**Fields \`ids\` generation**](#org035a59b)
    3.  [**Field type**](#field-type)
    4.  [**Question Grouping**](#orgd739459)
    5.  [**Display When**](#orgabf4c78)
    6.  [**Enable When**](#org9424633)
    7.  [**Calculated fields**](#org099c6d2)
    8.  [**Default (initial) values**](#orgc4c273a)
    9.  [**Repeated questions**](#org35986f6)
    10. [**Constraints**](#org044f833)
    11. [**Extraction**](#org784e051)
    12. [**Layout**](#orgef80d87)
    13. [**Prefill**](#org1c276e8)
    14. [**Form Pieces Reuse (subforms)**](#org080d869)
    15. [**Default form fields**](#org08abb1d)
    16. [**How to store in DB**](#org8b2e3df)
3.  [Form examples applying this rules](#org91bc139)
    1.  [**Vitals**](#org0e1a6ab)
    2.  [**PHQ2/PHQ9**](#org1e72920)


<a id="org07d7d94"></a>

# Goal

1.  Make simplified DSL
2.  Move from layered arch to only document structure
3.  Make it a sugar for SDC Questionnaire
4.  Convertible <-> FHIR Q/QR
5.  Store in DB as meta resource


<a id="org3079260"></a>

# Decisions


<a id="org2fb4829"></a>

## **Remove :keys on top level**

1.  Rename to \`:questions\` as map:

        LateVisitDocument
        {:type aidbox.sdc/form
         :questions {:question1 {...}}}

    -   semantic
    -   easy to work from code perspective
    -   hard to human read

2.  Rename to \`:fields\` as map:

3.  Rename to \`:fields\` as vector

        LateVisitDocument
        {:type aidbox.sdc/form
         :fields [{:name "ahahha"}
                  {}]}

    -   more human readable
    -   layout generation

4.  Rename to \`:items\` as vector
    -   similar to fhir questionnaire


<a id="org035a59b"></a>

## **How to identify fields**

[Discussion](https://github.com/HealthSamurai/forms-design/discussions/1)


```clojure
LateVisitDocument
{:type aidbox.sdc/form
:questions [{:label "Super basic question"
             :id :basic-question}]}
```

<a id="field-type"></a>

## **Field type**

[Discussion](https://github.com/HealthSamurai/forms-design/discussions/3)

<a id="orgd739459"></a>

## **Question Grouping**

[Discussion](https://github.com/HealthSamurai/forms-design/discussions/5)

<a id="orgabf4c78"></a>

## **Display When**

1.  As additional key in lisp

        VitalsForm
        {:type aidbox.sdc/form
         :fields [{:name "label"
                   :display-when (= (get-in [:patient :name]) "")}]}

    -   hard to store in DB

2.  As additinal key in fhirpath

        VitalsForm
        {:type aidbox.sdc/form
         :fields [{:name "label"
                   :display-when "patient.name = '123'"}]}

    -   familiar to fhir community


<a id="org9424633"></a>

## **Enable When**

1.  As additional key:

        PHQ2PHQ9
        {:type aidbox.sdc/form
         :fields [{:name "PHQ9 Score"
                   :id "phq-score"
                   :enable-when "false" ;; always disabled (readonly)
                   }
                  {:name "Do you have depression?"
                   :type "checkbox"
                   :id "phq2-depression"
                   :enable-when "this.feeling-tired = null"}]}


<a id="org099c6d2"></a>

## **Calculated fields**

1.  Additional field with rule on top level:

        VitalsForm
        {:type aidbox.sdc/form
         :rules {:bmi (+ (get :weight) (get :height))
                 :fields [{:label "BMI"
                           :name "bmi"}]}

    This is current approach:

    -   all rules in one place
    -   not scalable, can't state the rule for nested fields
        and grouped items

2.  Rule on the same level as field:

        VitalsForm
        {:type aidbox.sdc/form
         :fields [{:name "BMI"
                   :rule (+ (get :weight) (get :height))}]}

3.  Under :value key

        VitalsForm
        {:type aidbox.sdc/form
         :fields [{:name "patient name"
                   :value (get-in [:patinet :name])}]}


<a id="orgc4c273a"></a>

## **Default (initial) values**

1.  Allocated field for default value:

        VitalsDocument
        {:type aidbox.sdc/form
         :fields [{:name "Gender"
                   :initial-value {:display "Not selected"}
                   :options [{:display "Male"}
                             {:display "Female"}]}]}

2.  Under :value key

        VitalsDocument
        {:type aidbox.sdc/form
         :fields [{:name "Gender"
                   :value {:dispaly "Not selected"}]}

    ?:
    How to distingiush this fields from calculated fields


<a id="org35986f6"></a>

## **Repeated questions**

1.  With type group we can only add additional flag about repeating:

        VitalsForm
        {:type aidbox.sdc/form
         :fields [{:name "patient"}
                  {:name "blood pressure"
                   :type "group"
                   :repetablae true
                   :fields [{:name "systolic"
                             :type "number"}]}

2.  Every field can be repeatable

        VitalsDocument
        {:fields [{:name "temperature"
                   :repeats true
                   :minItems 1
                   :type "quantity"
                   :unit "C"}]}


<a id="org044f833"></a>

## **Constraints**

1.  each question type knows its constraints

        VitalsForm
        {:type aidbox.sdc/form
         :fields [{:name "weight"
                   :min 123}
                  {:name "blood pressure"
                   :repeats true
                   :minItems 1}]}


<a id="org784e051"></a>

## **Extraction**

1.  Template based

    Covers only Observation cases

        VitalsDocument
        {:type aidbox.sdc/form
         :fields [{:id "temperature"
                   :type "quantity"
                   :extraction "observation"
                   :coding [{:system "loinc" :code "112312"}]
                   }]}

    -   where to get other fields for observation (subject, effectiveDate)
    -   how to infer value type (or composite)

1.  Inline template

    VitalsDocument

        VitalsDocument
        {:type aidbox.sdc/form
         :fields [{:id "temp"
                   :type "quantity"
                   :extraction {:resourceType "Observation"
                                :valueQuantity {:unit "123" :value "this"}
                                :code {:system "loinc" :code "123123"}}}
                  ]}

    -   boilerplate


<a id="orgef80d87"></a>

## **Layout**

In simple case we have one column with all fields
What if I want to place my fields in a row?

1.  Special control type as 'group'

        {:type aidbox.sdc/form
         :fields [{:id "blood-pressure"
                   :widget "row"}

2.  Special key in group control

        {:type aidbox.sdc/form
         :fields [{:id "blood-pressure"
                   :type "group"
                   :direction "row"}]}

3.  Optional key in each widget telling about control size:

        {:type aibox.sdc/form
         :fields [{:id "blood-pressure"
                   :type "group"
                   :fields [{:id "systolic"
                             :size 6}
                            {:id "diastolic"
                             :size 6}]}
                  ]

    Whole page divided into equal 12 columns. By default
    each field takes 12 columns. But we can say, how many columns
    should take this field. Like, say systolic and diastolic:
    take 6 of 12 columns.

    <!-- This HTML table template is generated by emacs 29.1 -->
    <table border="1">
      <tr>
        <td align="left" valign="top">
          systolic
        </td>
        <td align="left" valign="top">
          diastolic
        </td>
      </tr>
    </table>

    And with this approach we can provide better mobile support.


<a id="org1c276e8"></a>

## **Prefill**

1.  Use prefill key:

    VitalsDocument
    {:type aidbox.sdc/form
     :fields [{:name "Weight"
               :id "weight"
               :prefill "Patient.observation(code='weight').valueQuantity.value"}]}


<a id="org080d869"></a>

## **Form Pieces Reuse (subforms)**

1.  Not use form pieces
    a lot of boilerplate code

2.  Special type for field template

        LL358-3
        {:type aidbox.sdc/template
         :name "some name here"
         :id "paste id here"
         :extraction {:resourceType "Observation"
                      :valueCoding "this.value"
                      :code "this.coding.0"}
         :options [{:display "Not at all" :code "LA6568-5" :score 0}
                   {:display "Several days" :code "LA6568-5" :score 1}
                   {:display "More than half the days" :code "LA6568-5" :score 2}
                   {:display "Nearly every day" :code "LA6568-5" :score 3}]}

    Then use it via "*use*"|"*template*" key. Keys will be merged

        PHQ2Form
        {:type aidbox.sdc/form
         :fields [{:use LL358-3
                   :name "Feeling bad about yourself"
                   :id "feeling-bad"
                   :code [...]}
                  {:template LL358-3
                   :name "Thoughts you would be better off dead"
                   :id "feeling-bad"
                   :code [..]}


<a id="org08abb1d"></a>

## **Default form fields**

Current implementation has several always-presented fields: \`author\`, \`patient\`, \`encounter\`


<a id="org8b2e3df"></a>

## **How to store in DB**

1.  Stored form should be the same as DSL?
2.  Should we store meta data (as in QuestionnaireResponse: question text, code)?


<a id="org91bc139"></a>

# Form examples applying this rules


<a id="org0e1a6ab"></a>

## **Vitals**

    VitalsDocument
    {:type aidbox.sdc/form
     :fields [{:name "Temperature"
               :id "temp"
               :repeats true
               :unit "F"
               :type "number"
               :min 86
               :max 105}
              {:name "Blood Pressure"
               :id "blood-pressure"
               :repeats true
               :fields [{:name "BP Sys" :id "systolic"
                         :unit "mmHg"
                         :min 40
                         :max 300}
                        {:name "BP dias"
                         :id "diastolic"
                         :unit "mmHg"
                         :min 20
                         :max 220}
                        {:name "Arm"
                         :id "arm"
                         :options [{:display "Biceps left"
                                    :code "LA11158-5"}
                                   {:display "Biceps Right"
                                    :code "LA11159-3"}]}
                        {:name "Position"
                         :id "position"
                         :options [{:display "Sitting"
                                    :code "LA11868-9"}
                                   {:display "Lying"
                                    :code "LA11868-9"}
                                   {:display "Standing"
                                    :code "LA11868-9"}]}]}
              {:name "Respiratory rate"
               :id "bpm"
               :repeats true
               :unit "bpm"
               :min 6 :max 60}
              {:name "Saturation"
               :id "saturation"
               :unit "SaO2 % PulseOx"
               :repeats true
               :min 60 :max 100}
              {:name "Heart Rate"
               :id "heart-rate"
               :unit "bpm"
               :repeats true
               :min 30 :max 250}
              {:name "BMI"
               :id "bmi"
               :type "number"
               :enable "false"
               :value "globalThis.weight.value / globalThis.height.value"}
              {:name "Weight"
               :id "weight"
               :unit "kg"}
              {:name "Height"
               :id "height"
               :unit "cm"}]}


<a id="org1e72920"></a>

## **PHQ2/PHQ9**

    LL358-3
    {:type aidbox.sdc/template
     :id "replace this"
     :name "replace this"
     :options [{:display "Not at all" :code "LA6568-5" :score 0}
               {:display "Several days" :code "LA6569-3" :score 1}
               {:display "More than half the days" :code "LA6570-1" :score 2}
               {:display "Nearly every day" :code "LA6571-9" :score 3}]}

    PHQ2PHQ9
    {:type "aidbox.sdc/form"
     :fields [{:type "label"
               :label "Over the past 2 weeks, how often have you been bothered by:"}
              {:template LL358-3
               :name "Feeling down, depressed or hopeless"
               :id "feeling-down"}
              {:template LL358-3
               :name "Little interest or pleasure in doing things"
               :id "little-interest"}
              {:type "group"
               :id "phq9-questions"
               :enable "this.little-interest + this.feeling-down >= 3"
               :fields  [{:template LL358-3
                          :name "Feeling bad about yourself"
                          :id "feeling-bad"}
                         {:template LL358-3
                          :name "Thoughts that you would be better off dead"
                          :id "thoughts"}
                         {:template LL358-3
                          :name "Poor appetite or overeating"
                          :id "poor-appetite"}
                         {:template LL358-3
                          :name "Trouble concentrating on things"
                          :id "trouble-concentrating"}

                         {:template LL358-3
                          :name "Trouble falling or staying asleep"
                          :id "trouble-sleep"}

                         {:template LL358-3
                          :name "Feeling tired or having little energy"
                          :id "feeling-tired"}
                         {:template LL358-3
                          :name "Moving or speaking so slowly"
                          :id "moving-slowly"}]}
              {:id "total-score"
               :name "PHQ2/9 Total Score"
               :value "this.feeling-down.score + this.little-interest.score + this.feeling-bad.score"}
              {:id "score-interp"
               :name "Score Interpretation"
               :value "if(this.feeling-down.score) some value"}
              ]}