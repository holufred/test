ClearCollect(
    colPlanAttributes,
    ForAll(
        Table(ParseJSON(GetPlanAttributes.Run().result)),
        {
            Year: Text(Value.Year),
            Quarter: Text(Value.Quarter),
            Participant: Text(Value.Participant),
            FileName: Text(Value.'[File Name]')
        }
    )
)
