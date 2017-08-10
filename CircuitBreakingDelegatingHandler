using System;
using System.Collections.Generic;
using System.Linq;
using System.Net.Http;
using System.Text;
using System.Threading.Tasks;
using Polly;
using Polly.Timeout;
using System.Threading;
using Polly.CircuitBreaker;

namespace ConsoleApplication2
{
    public class CircuitBreakingDelegatingHandler : DelegatingHandler
    {
        private readonly int _exceptionsAllowedBeforeBreaking;
        private readonly TimeSpan _durationOfBreak;
        private readonly Policy _circuitBreakerPolicy;
        private readonly TimeoutPolicy _timeoutPolicy;
        public CircuitBreakingDelegatingHandler(HttpMessageHandler innerHandler) : base(innerHandler)
        {
            this._exceptionsAllowedBeforeBreaking = 3;
            this._durationOfBreak = TimeSpan.FromSeconds(1);
            this._circuitBreakerPolicy = Policy.Handle<Exception>()
                .Or<HttpRequestException>()
                .Or<TimeoutRejectedException>()
                .Or<TimeoutException>()
                .CircuitBreakerAsync(exceptionsAllowedBeforeBreaking: _exceptionsAllowedBeforeBreaking, durationOfBreak: _durationOfBreak,
                onBreak: (ex, breakDelay) =>
                {
                    Console.WriteLine($".Breaker logging: Breaking the circuit for {breakDelay.TotalMilliseconds}ms! ex={ex.Message}");
                },
                onReset: () =>
                {
                    Console.WriteLine($".Breaker logging: Call ok! Closed the circuit again.");
                },
                onHalfOpen: () =>
                {
                    Console.WriteLine($".Breaker logging: Half-open; next call is a trial.");
                });
            _timeoutPolicy = Policy.TimeoutAsync(TimeSpan.FromMilliseconds(100), TimeoutStrategy.Optimistic);
        }

        protected override async Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
        {            
            try
            {
                var policyResult = await Policy.WrapAsync(_circuitBreakerPolicy, _timeoutPolicy).ExecuteAndCaptureAsync<HttpResponseMessage>(() =>
                 {
                     return base.SendAsync(request, cancellationToken);
                 });
                if (policyResult != null && policyResult.FinalException != null)
                {
                    return await Task<HttpResponseMessage>.FromResult(new HttpResponseMessage() { StatusCode = System.Net.HttpStatusCode.InternalServerError });
                }
                return policyResult.Result;
            }
            catch (BrokenCircuitException ex)
            {
                Console.WriteLine($"Reached to allowed number of exceptions. Circuit is open. AllowedExceptionCount: {_exceptionsAllowedBeforeBreaking}, DurationOfBreak: {_durationOfBreak} ex={ex.Message}");                
            }
            catch (HttpRequestException)
            {
                Console.WriteLine($"HttpRequestException");
            }
            return await Task<HttpResponseMessage>.FromResult(new HttpResponseMessage() { StatusCode = System.Net.HttpStatusCode.InternalServerError });
        }
        private static bool IsTransientFailure(HttpResponseMessage response)
        {
            return response.StatusCode >= System.Net.HttpStatusCode.InternalServerError;
        }
    }

}
